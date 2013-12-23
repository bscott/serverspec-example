require 'rake'
require 'rspec/core/rake_task'
require 'xpool'
require 'colorize'
require 'json'

$HOSTS    = "./hosts"           # List of all hosts
$REPORTS  = "./reports"         # Where to store JSON reports
$PARALLEL = 10                  # How many parallel tasks should we run?

# Return all roles of a given host
def roles(host)
  roles = [ "all" ]
  case host
  when /^blm-web-/
    roles << "web"
  when /^blm-memc-/
    roles << "memcache"
  when /^blm-lb-/
    roles << "lb"
  when /^blm-bigdata-/
    roles << "bigdata"
  when /^blm-proxy-/
    roles << "proxy"
  end
  return roles
end

# Run RSpec in the background. It is important to define this before
# creating a pool.
class BackgroundServerspecTask

  def run(verbose, target, command, json)
    ENV['TARGET_HOST'] = target
    system(command)
    status(target, json) if verbose
  end

  # Display status of a test from its JSON output
  def status(target, json)
    begin
      out = JSON.parse(File.read(json))
      summary = out["summary"]
      total = summary["example_count"]
      failures = summary["failure_count"]
      if failures > 0 then
        print ("[%-3s/%-4s] " % [failures, total]).yellow, target, "\n"
      else
        print "[OK      ] ".green, target, "\n"
      end
    rescue Exception => e
      print "[ERROR   ] ".red, target, " (#{e.message})", "\n"
    end
  end

end

# Special version of RakeTask that will be executed using a process
# pool. This process pool is handled by xpool.
class ServerspecTask < RSpec::Core::RakeTask

  attr_accessor :target

  @@pool = XPool.new $PARALLEL

  def self.shutdown
    @@pool.shutdown
  end

  def run_task(verbose)
    json = "#{$REPORTS}/current/#{target}.json"
    @rspec_opts = ["--format", "json", "--out", json]
    command = spec_command
    @@pool.schedule BackgroundServerspecTask.new, verbose, target, command, json
  end
end

hosts = File.foreach(ENV["HOSTS"] || $HOSTS)
  .map { |line| line.strip }
  .map do |host|
  {
    :name => host.strip,
    :roles => roles(host.strip),
  }
end

desc "Run serverspec to all hosts"
task :spec => "check:server:all"

namespace :check do

  # Per server tasks
  namespace :server do
    desc "Run serverspec to all hosts"
    task :all => hosts.map { |h| h[:name] }
    hosts.each do |host|
      desc "Run serverspec to host #{host[:name]}"
      ServerspecTask.new(host[:name].to_sym) do |t|
        t.target = host[:name]
        t.pattern = './spec/{' + host[:roles].join(",") + '}/*_spec.rb'
      end
    end
  end

  # Per role tasks
  namespace :role do
    roles = hosts.map {|h| h[:roles]}
    roles = roles.flatten.uniq
    roles.each do |role|
      desc "Run serverspec to role #{role}"
      task "#{role}" => hosts.select { |h| h[:roles].include? role }.map {
        |h| "check:server:" + h[:name]
      }
    end
  end
end

namespace :reports do
  desc "Clean up old reports"
  task :clean do
    FileUtils.rm_rf "#{$REPORTS}/current"
  end

  desc "Build final report"
  task :build do
    now = Time.now
    fname = "#{$REPORTS}/run-%s.json" % [ now.strftime("%Y-%m-%dT%H:%M:%S") ]
    File.open(fname, "w") { |f|
      f.puts JSON.generate(FileList.new("#{$REPORTS}/current/*.json").sort.map { |j|
                             content = File.read(j).strip
                             {
                               :hostname => File.basename(j, ".json"),
                               :results => JSON.parse(content.empty? ? "{}" : content)
                             }
                           }.to_a)
    }
  end
end

check_tasks = Rake.application.top_level_tasks.select { |task| task.start_with?("check:") }
if not check_tasks.empty? then
  # Before starting, cleanup reports
  Rake::Task[check_tasks.first].enhance [ "reports:clean" ]

  # Build final report
  Rake::Task[check_tasks.last].enhance do
    ServerspecTask.shutdown
    Rake::Task["reports:build"].invoke
  end
end

Kernel.at_exit do
  ServerspecTask.shutdown
end
