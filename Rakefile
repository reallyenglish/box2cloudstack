# Generated by `infrataster init`

require "English"
require "rake"
require "fileutils"
require "yaml"

template_dir = "templates"

def rake(path:, args:)
  puts "Running rake #{args}"
  Bundler.with_clean_env do
    Dir.chdir(path) do
      sh "bundle exec rake #{args}"
    end
  end
end

inventory_file = "#{template_dir}/inventories/default.yml"
inventory = YAML.load_file(inventory_file)

inventory["all"]["hosts"].each_key do |name|
  namespace name do
    output_dir = "output-#{name}"
    desc "initialize build directory"
    task :setup do |_t|
      puts "Creating #{output_dir}"
      Dir.mkdir(output_dir)
      puts "Copying template to #{output_dir}"
      FileUtils.cp_r("#{template_dir}/.", output_dir)
      rake path: output_dir, args: "setup"
    end

    desc "do_build #{name}"
    task :do_build do |_t|
      rake path: output_dir, args: "#{name}:build"
      puts "Copying #{output_dir}/#{name}.ova to #{name}.ova"
      FileUtils.cp "#{output_dir}/#{name}.ova", "#{name}.ova"
    end

    desc "clean #{output_dir}"
    task :clean do |_t|
      rake path: output_dir, args: "clean"
      puts "Removing #{output_dir}"
      FileUtils.rm_rf [output_dir], :secure => true
    end

    desc "build #{name} and clean #{output_dir}"
    task :build => [:setup, :do_build, :clean]
  end
end

desc "build all"
all_targets = inventory["all"]["hosts"].keys.sort.map {|i| "#{i}:build" }
task :build => all_targets do |_t|
end

desc "clean everything"
task :clean do |_t|
  begin
    Dir.glob("output-*").each do |dir|
      rake :path => dir, :args => "clean"
    end
  ensure
    puts "Removing #{Dir.glob("output-*")}"
    FileUtils.rm_rf Dir.glob("output-*"), :secure => true
    puts "Removing #{Dir.glob("*.ova")}"
    FileUtils.rm_rf Dir.glob("*.ova"), :secure => true
  end
end