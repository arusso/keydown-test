# iptables and NRPE management

!SLIDE

## Managing iptables and NRPE in Puppet
### Aaron Russo <arusso@berkeley.edu>

!SLIDE

## Introduction

To really leverage Puppet, we need to work with objects within Puppet, and step
away from the copy-a-file mindset.

To that end, I've created some puppet modules to manage iptables and nrpe rules,
taking care to keep the on-disk configuration to as similar as possible to how
we already do thing.

This is their story.

!SLIDE left

## Outline

 1. iptables
     1. How are we handling them now?
     1. How can we handle them better?
     1. But wait, there's more...
 1. NRPE
     1. What are we doing now?
     1. What are we fixing?

!SLIDE left

## iptables / What are we doing now?

### Overview

 * single base firewall class
 * firewall rulesets are in static files
 * each host needing additional rules requires a completely new file
 * ip6tables is supported, but doubles the number of files to edit

### Stats

 * Hosts: 76
 * Firewalls (IPv4 / IPv6): 55 / 5

!SLIDE

## iptables / What are the issues, how do we fix them?

The main course of this presentation -- why exactly do we need this?

### configuration sprawl/duplication of data
 
 iptables rules are now defined types in puppet, so can be easily wrapped in
 classes and reused.

Example:
```ruby
# modules eu/manifests/firewall/web.pp
class eu::firewall::web {
  iptables::rule { 'eu::firewall::web - allow public web access':
    comment          => 'eu::firewall::web - allow web access',
    order            => 200,
    protocol         => 'tcp',
    destination_port => [ '80', '443' ]
  }
}

# modules/app/manifess/supadupa.pp
class app::supadupa {
  include eu::firewall::base
  if $::hostname =~ /web/ { include eu::firewall::web }
}

# modules/app/manifests/snafupa.pp
class app::snafupa {
  include eu::firewall::base
  if $::hostname =~ /web/ { include eu::firewall::web }
}

# manifests/eu-snafupa-web-01.ist.berkeley.edu.pp
node 'eu-snafupa-web-01.ist.berkeley.edu' {
  include app::snafupa
}

# manifests/eu-supadupa-web-01.ist.berkeley.edu.pp
node 'eu-supadupa-web-01.ist.berkeley.edu' {
  include app::supadupa
}
```

!SLIDE

## What problems are we addressing?

### minimize amount of time it takes to audit and make changes

 with 55 rulesets, it would take all of us a good chunk of time auditing the
 rules that are applied to those hosts.  if all of the rules are in puppet and
 rulesets are reused, we only need to audit a handful of rulesets.
