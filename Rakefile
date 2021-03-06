require 'rubygems'
require 'bundler/setup'
require 'date'
require 'fog'

#############################################################################
#
# Helper functions
#
#############################################################################

def name
  @name ||= Dir['*.gemspec'].first.split('.').first
end

def version
  line = File.read("lib/#{name}.rb")[/^\s*VERSION\s*=\s*.*/]
  line.match(/.*VERSION\s*=\s*['"](.*)['"]/)[1]
end

def date
  Date.today.to_s
end

def rubyforge_project
  name
end

def gemspec_file
  "#{name}.gemspec"
end

def gem_file
  "#{name}-#{version}.gem"
end

def replace_header(head, header_name)
  head.sub!(/(\.#{header_name}\s*= ').*'/) { "#{$1}#{send(header_name)}'"}
end

#############################################################################
#
# Standard tasks
#
#############################################################################

task :default => :test

task :examples do
  sh("export FOG_MOCK=false && bundle exec shindont examples")
  # some don't provide mocks so we'll leave this out for now
  # sh("export FOG_MOCK=true  && bundle exec shindont examples")
end

task :test => :examples do
  sh("export FOG_MOCK=true  && bundle exec spec spec") &&
  sh("export FOG_MOCK=true  && bundle exec shindont tests") &&
  sh("export FOG_MOCK=false && bundle exec spec spec") &&
  sh("export FOG_MOCK=false && bundle exec shindont tests")
end

desc "Generate RCov test coverage and open in your browser"
task :coverage do
  require 'rcov'
  sh "rm -fr coverage"
  sh "rcov test/test_*.rb"
  sh "open coverage/index.html"
end

require 'rake/rdoctask'
Rake::RDocTask.new do |rdoc|
  rdoc.rdoc_dir = 'rdoc'
  rdoc.title = "#{name} #{version}"
  rdoc.rdoc_files.include('README*')
  rdoc.rdoc_files.include('lib/**/*.rb')
end

desc "Open an irb session preloaded with this library"
task :console do
  sh "irb -rubygems -r ./lib/#{name}.rb"
end

#############################################################################
#
# Packaging tasks
#
#############################################################################

task :release => :build do
  unless `git branch` =~ /^\* master$/
    puts "You must be on the master branch to release!"
    exit!
  end
  sh "gem install pkg/#{name}-#{version}.gem"
  sh "git commit --allow-empty -a -m 'Release #{version}'"
  sh "git tag v#{version}"
  sh "git push origin master"
  sh "git push origin v#{version}"
  sh "gem push pkg/#{name}-#{version}.gem"
  Rake::Task[:docs].invoke
  Rake::Task[:changelog].invoke
end

task :build => :gemspec do
  sh "mkdir -p pkg"
  sh "gem build #{gemspec_file}"
  sh "mv #{gem_file} pkg"
end

task :gemspec => :validate do
  # read spec file and split out manifest section
  spec = File.read(gemspec_file)

  # replace name version and date
  replace_header(spec, :name)
  replace_header(spec, :version)
  replace_header(spec, :date)
  #comment this out if your rubyforge_project has a different name
  replace_header(spec, :rubyforge_project)

  File.open(gemspec_file, 'w') { |io| io.write(spec) }
  puts "Updated #{gemspec_file}"
end

task :validate do
  libfiles = Dir['lib/*'] - ["lib/#{name}.rb", "lib/#{name}"]
  unless libfiles.empty?
    puts "Directory `lib` should only contain a `#{name}.rb` file and `#{name}` dir."
    exit!
  end
  unless Dir['VERSION*'].empty?
    puts "A `VERSION` file at root level violates Gem best practices."
    exit!
  end
end

task :changelog do
  timestamp = Time.now.utc.strftime('%m/%d/%Y')
  sha = `git log | head -1`.split(' ').last
  changelog = ["#{version} #{timestamp} #{sha}"]
  changelog << ('=' * changelog[0].length)
  changelog << ''

  last_sha = `cat changelog.txt | head -1`.split(' ').last
  shortlog = `git shortlog #{last_sha}..HEAD`
  changes = {}
  for line in shortlog.split("\n")
    if line =~ /^\S/
      committer = line.split(' (', 2).first
    elsif line =~ /^\s*((Merge.*)|(Release.*))?$/
      # skip empty lines, Merge and Release commits
    else
      unless line[-1..-1] == '.'
        line << '.'
      end
      line.gsub!(/^\s*\[([^\]]*)\] /, '')
      tag = $1 || 'misc'
      changes[tag] ||= []
      changes[tag] << (line << ' thanks ' << committer)
    end
  end

  for tag in changes.keys.sort
    changelog << ('[' << tag << ']')
    for commit in changes[tag]
      changelog << ('  ' << commit)
    end
    changelog << ''
  end

  `echo "#{changelog.join("\n")}" | pbcopy`
  p 'changelog copied to clipboard'
end

task :docs do
  # build the docs locally
  sh "jekyll docs docs/_site"

  # connect to storage provider
  Fog.credential = :geemus
  storage = Fog::Storage.new(:provider => 'AWS')
  directory = storage.directories.new(:key => 'fog.io')

  # write web page files to versioned 'folder'
  for file_path in Dir.glob('docs/_site/**/*')
    next if File.directory?(file_path)
    file_name = file_path.gsub('docs/_site/', '')
    key = '' << version << '/' << file_name
    Formatador.redisplay(' ' * 128)
    Formatador.redisplay('Uploading ' << key)
    if File.extname(file_name) == '.html'
      # rewrite links with version
      body = File.read(file_path)
      body.gsub!(/vX.Y.Z/, 'v' << version)
      body.gsub!(/='\//, %{='/} << version << '/')
      body.gsub!(/="\//, %{="/} << version << '/')
      content_type = 'text/html'
      directory.files.create(
        :body         => redirecter(key),
        :content_type => 'text/html',
        :key          => 'latest/' << file_name,
        :public       => true
      )
    else
      body = File.open(file_path)
      content_type = nil # leave it up to mime-types
    end
    directory.files.create(
      :body         => body,
      :content_type => content_type,
      :key          => key,
      :public       => true
    )
  end
  Formatador.redisplay(' ' * 128)
  Formatador.redisplay('Uploaded docs/_site')

  # write rdoc files to versioned 'folder'
  Rake::Task[:rdoc].invoke
  for file_path in Dir.glob('rdoc/**/*')
    next if File.directory?(file_path)
    file_name = file_path.gsub('rdoc/', '')
    key = '' << version << '/rdoc/' << file_name
    Formatador.redisplay(' ' * 128)
    Formatador.redisplay('Uploading ' << key)
    directory.files.create(
      :body         => File.open(file_path),
      :key          => key,
      :public       => true
    )
  end
  Formatador.redisplay(' ' * 128)
  directory.files.create(
    :body         => redirecter("#{version}/rdoc/index.html"),
    :content_type => 'text/html',
    :key          => 'latest/rdoc/index.html',
    :public       => true
  )
  Formatador.redisplay('Uploaded rdoc')

  # write base index with redirect to new version
  directory.files.create(
    :body         => redirecter(version),
    :content_type => 'text/html',
    :key          => 'index.html',
    :public       => true
  )

  Formatador.display_line
end

def redirecter(path)
  redirecter = <<-HTML
<!doctype html>
<head>
<title>fog</title>
<meta http-equiv="REFRESH" content="0;url=http://fog.io/#{path}">
</head>
<body>
  <a href="http://fog.io/#{path}">redirecting to lastest (#{path})</a>
</body>
</html>
HTML
end