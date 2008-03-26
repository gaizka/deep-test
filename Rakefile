require 'rake'
require 'rake/testtask'
require 'rake/rdoctask'
require 'rake/gempackagetask'
require 'rake/contrib/sshpublisher'
$LOAD_PATH << File.dirname(__FILE__) + "/lib"
require "deep_test/rake_tasks"

task :default => %w[test spec failing_test deep_test deep_spec run_distributed run_distributed_with_worker_down test_rails_project]

Rake::TestTask.new do |t|
  t.pattern = "test/**/*_test.rb"
  t.libs += ['test', 'lib']
end

DeepTest::TestTask.new :deep_test do |t|
  t.number_of_workers = 2
  t.pattern = "test/**/*_test.rb"
end

DeepTest::TestTask.new(:distributed_test) do |t|
  t.number_of_workers = 2
  t.pattern = "test/**/*_test.rb"
  t.distributed_server = "druby://localhost:8000"
  t.sync_options = {:source => File.dirname(__FILE__), 
                    :local => true,
                    :rsync_options => "--exclude=.svn"}
end

def rspec_present?
  defined?(Spec)
end

if rspec_present?
  Spec::Rake::SpecTask.new(:spec) do |t|
    t.spec_files = FileList['spec/**/*_spec.rb']
  end
  
  Spec::Rake::SpecTask.new(:deep_spec) do |t|
    t.spec_files = FileList['spec/**/*_spec.rb']
    t.deep_test :number_of_workers => 2
  end

  Spec::Rake::SpecTask.new(:distributed_spec) do |t|
    t.spec_files = FileList['spec/**/*_spec.rb']
    t.deep_test :number_of_workers => 2, 
                :distributed_server => "druby://localhost:8000",
                :sync_options => {:source => File.dirname(__FILE__), 
                                  :local => true,
                                  :rsync_options => "--exclude=.svn"}
  end
end


DeepTest::TestTask.new(:distribute_tests_to_minis) do |t|
  t.number_of_workers = 2
  t.pattern = "test/**/*_test.rb"
  t.distributed_server = "druby://alpha.local:8000"
  t.sync_options = {:source => File.dirname(__FILE__), 
                    :username => "tworker",
                    :password => "thought",
                    :rsync_options => "--exclude=.svn"}
end

task :run_distributed do |t|
  begin
    FileUtils.mkdir('/tmp/mirror_1') unless File.exist?('/tmp/mirror_1')
    FileUtils.mkdir('/tmp/mirror_2') unless File.exist?('/tmp/mirror_2')

    mirror_1_pid = fork do
      exec "ruby bin/deep_test mirror_server --uri druby://localhost:8001 --mirror_base_path /tmp/mirror_1"
    end

    mirror_2_pid = fork do
      exec "ruby bin/deep_test mirror_server --uri druby://localhost:8002 --mirror_base_path /tmp/mirror_2"
    end

    master_pid = fork do
      exec "ruby bin/deep_test master_mirror_server --uri druby://localhost:8000 druby://localhost:8001 druby://localhost:8002"
    end

    sleep 1

    Rake::Task[:distributed_test].invoke
    Rake::Task[:distributed_spec].invoke

    sh "ruby bin/deep_test test_throughput druby://localhost:8000 20"
  ensure
    Process.kill('TERM', master_pid) if master_pid rescue nil
    Process.kill('TERM', mirror_1_pid) if mirror_1_pid rescue nil
    Process.kill('TERM', mirror_2_pid) if mirror_2_pid rescue nil
    Process.waitall
  end

  sleep 1
end

task :run_distributed_with_worker_down do |t|
  begin
    FileUtils.mkdir('/tmp/mirror_1') unless File.exist?('/tmp/mirror_1')

    mirror_1_pid = fork do
      exec "ruby bin/deep_test mirror_server --uri druby://localhost:8001 --mirror_base_path /tmp/mirror_1"
    end

    # don't start worker 2

    master_pid = fork do
      exec "ruby bin/deep_test master_mirror_server --uri druby://localhost:8000 druby://localhost:8001 druby://localhost:8002"
    end

    sleep 1

    # Use throughput to make sure we can run tests
    sh "ruby bin/deep_test test_throughput druby://localhost:8000 20"
  ensure
    Process.kill('TERM', master_pid) if master_pid rescue nil
    Process.kill('TERM', mirror_1_pid) if mirror_1_pid rescue nil
    Process.waitall
  end
end

task :failing_test do
  command = "rake --rakefile test/failing.rake 2>&1"
  puts command
  `#{command}`
  if $?.success?
    puts "F"
    fail "****\ntest/failing.rake should have failed\n****"
  else
    puts "."
  end
end

task :test_rails_project do
  Dir.chdir('sample_rails_project') do |path|
    system "rake test:units"
    system "rake deep_test"
  end
end

desc "Generate documentation"
Rake::RDocTask.new(:rdoc) do |rdoc|
  rdoc.rdoc_dir = "doc"
  rdoc.title    = "DeepTest"
  rdoc.options << '--line-numbers'
  rdoc.rdoc_files.include(
    'README', 
    'CHANGELOG', 
    'lib/deep_test/null_worker_listener.rb',
    'lib/deep_test/database/*.rb'
  )
end

desc "Upload RDoc to RubyForge"
task :publish_rdoc => [:rdoc] do
  Rake::SshDirPublisher.new("dcmanges@rubyforge.org", "/var/www/gforge-projects/deep-test", "doc").upload
end

Gem::manage_gems

specification = Gem::Specification.new do |s|
  s.platform = Gem::Platform::RUBY
	s.name   = "deep_test"
  s.summary = "DeepTest runs tests in multiple processes."
	s.version = "1.2.0"
	s.author = "anonymous z, Dan Manges, David Vollbracht"
	s.description = s.summary
	s.email = "daniel.manges@gmail.com"
  s.homepage = "http://deep-test.rubyforge.org"
  s.rubyforge_project = "deep-test"
  s.executables << "deep_test"

  s.has_rdoc = true
  s.extra_rdoc_files = ['README', 'CHANGELOG']
  s.rdoc_options << '--title' << "DeepTest" << '--main' << 'README' << '--line-numbers'

  s.autorequire = "deep_test"
  s.files = FileList['{lib,script,test}/**/*.{rb,rake}', 'README', 'CHANGELOG', 'Rakefile'].to_a
end

Rake::GemPackageTask.new(specification) do |package|
  package.need_zip = true
  package.need_tar = true
end

Rake::Task[:gem].prerequisites.unshift :deep_test
Rake::Task[:gem].prerequisites.unshift :test

task :tar do
  system "tar zcf pkg/deep_test.tar.gz --exclude=.svn --exclude='*.tar.gz' --exclude='*.gem' --directory=.. deep_test"
end
