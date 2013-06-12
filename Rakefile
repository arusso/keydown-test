task :default => [:build,:preview]

task :build do
  sh "keydown slides slides.md"
end

task :preview do
  sh "open slides.html"
end
