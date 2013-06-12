# iptables and NRPE management

!SLIDE

## Managing iptables and NRPE in Puppet
### Aaron Russo <arusso@berkeley.edu>

!SLIDE

CFEngine likes files, Puppet likes types

!SLIDE

<p>The way we manage iptables and nrpe now fits the CFEngine model</p>

!SLIDE

<p>Here's how we define where we pickup iptables files:</p>
@@@ ruby
# modules/sys_iptables_rhfw/manifests/init.pp
... snip ...
file { '/etc/sysconfig/iptables':
  mode   => '0400',
  owner  => 'root',
  group  => 'root',
  source => [ 'puppet:///host/etc/sysconfig/iptables',
              "puppet:///modules/sys_iptables_rhfw/etc/sysconfig/iptables-$location",
              'puppet:///modules/sys_iptables_rhfw/etc/sysconfig/iptables' ]
}
... snip ...
@@@

<p>And here's NRPE configurations</p>
@@@ ruby
# modules/sys_nagios_std/manifests/init.pp
... snip
file { '/etc/nrpe.d/local.cfg':
  owner => 'root',
  group => 'root',
  mode  => '0444',
  source => [ 'puppet:///host/etc/nrpe.d/local.cfg',
              'puppet:///modules/sys_nagios_std/etc/nrpe.d/local.cfg',
            ]
}
... snip ...
@@@

!SLIDE

<p>"Hold up, what's the matter with that?"</p>

!SLIDE

1. It leads to a ton of duplicated configuration</p>

1. We currently have 55 individual firewall rulesets for 76 hosts.

1. There's no common location for certain configuration items like in CFEngine.

1. This is a huge pain spot if we ever need to make no-trivial updates across the
board.

1. It's not **immediately** clear where iptables rules are located.  They can be
in one of more than 3 possible places.

!SLIDE

<p>"So, how do we fix this?"</p>

!SLIDE

By using dedicated puppet **types** to declare **resources** that we can
group together within **classes** for reuse

Let's see what that looks like.

!SLIDE

###### Here's a class with 3 resources we can reference as `eu::firewall::base`
@@@ ruby
  # modules/eu/manifests/firewall/base.pp
  class eu::firewall::base {
    iptables::rule { 'base-allow-lo-int':
      comment            => 'allow lo traffic',
      order              => 250,
      incoming_interface => 'lo',
    }

    iptables::rule { 'ssh-192.168.0.0/24-admin-net':
      comment          => 'allow ssh from admin network',
      order            => 100,
      protocol         => 'tcp',
      destination_port => '22',
    }

    iptables::rule { 'related, established traffic':
      comment          => 'allow related, established traffic',
      order            => 050,
      protocol         => 'tcp',
      destination_port => '22',
    }
  }
@@@

!SLIDE

###### Here's a class with a 1 resource we can reference as `eu::firewall::web`
@@@ ruby
  # modules eu/manifests/firewall/web.pp
  class eu::firewall::web {
    iptables::rule { 'eu::firewall::web - allow public web access':
      comment          => 'eu::firewall::web - allow web access',
      order            => 200,
      protocol         => 'tcp',
      destination_port => [ '80', '443' ]
    }
  }
@@@

###### Here's a class with 2 included classes and 1 resource we can reference as `app::snafu`
@@@ ruby
  # modules/app/manifests/snafu.pp
  class app::snafu {
    include eu::firewall::base
    include eu::firewall::web

    iptables::rule { 'allow smtp incoming from admin networks':
      comment          => 'ticket#12345 - allow ingress smtp',
      protocol         => 'tcp',
      destination_port => [ '25', '587' ],
      order            =>'450,
      network          => $eu::firewall::admin_networks,
      source           => [ '2607::1/120', '10.0.1.0/24' ],
    }
  }
@@@

!SLIDE

###### Finally, let's use our application class
@@@ ruby
# manifests/snafu-prod-01.pp
node 'snafu.example.com' {
  ... snip ...

  include app::snafu

  ... snip ...
}
@@@

###### Or we can just call the firewall class(es) directly.
@@@ ruby
node 'fubar.example.com' {
  ... snip ...

  include eu::firewall::base

  ... snip ...
}
@@@

!SLIDE

### IPv6 Support

* It's all baked in -- you shouldn't need to do anything special to use it.

* if you don't specify a source or destination ip, nor a protocol version, an
iptables AND ip6tables rule(s) will be generated.

* if you specify only addresses within a particular protocol version, the rule
will only be created for that protocol version.

* if you specify addresses from both protocol versions, the rule will generate
equivalent rules for both iptables and ip6tables with the appropriate addresses.

!SLIDE

"Wait, but what about NRPE?"

!SLIDE

NRPE is handled in a very similar fashion, using the `nrpe` class, and the
`nrpe::command` type.

!SLIDE

###### Here the class that sets up NRPE behind xinetd for us
@@@ ruby
# modules/eu/manifests/nrpe.pp
class eu::nrpe {
  class { '::nrpe':
    xinetd        => true,
    allowed_hosts => $eu::firewall::nrpe_monitors,
  }

  $plugindir = '/usr/lib64/nagios/plugins'
}
@@@

###### And here's a class that defines a few of our commands
@@@ ruby
# modules/eu/manifests/nrpe/base.pp
class eu::nrpe::base (
  $users_warn = 8,
  $users_crit = 12,
  $load_warn = '15,10,5',
  $load_crit = '30,25,20'
) {
  include eu::nrpe

  nrpe::command { 'check_users':
    command => "${plugindir}/check_users -w ${users_warn} -c ${users_crit}",
  }

  nrpe::command { 'check_load':
    command => "${plugindir}/check_load -w ${load_warn} -c ${load_crit}",
  }
}
@@@

!SLIDE

###### Now we'll apply this to a host along with a host-specific command
@@@ ruby
# manifests/snafu.example.com
node 'snafu.example.com' {
  ... snip ...

  class { 'eu::nrpe::base':
    users_warn => 99,
    users_crit => 100,
  }

  nrpe::command { 'check_snafu':
    command => "/usr/local/bin/check_snafu -w 10 -c 15',
  }

  ... snip ...
}
@@@

!SLIDE

Files are generated under /etc/nrpe.d/<command name>

From our previous examples:

@@@ text
# /etc/nrpe.d/check_snafu.cfg
command[check_snafu]=/usr/local/bin/check_snafu -w 10 -c 15

# /etc/nrpe.d/check_users.cfg
command[check_users]=/usr/lib64/nagios/plugins/check_users -w 99 -c 100

# /etc/nrped/check_load.cfg
command[check_load]=/usr/lib64/nagios/plugins/check_load -w 15,10,5 -c 30,25,20
@@@
