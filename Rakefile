# Generated by `infrataster init`

require "English"
require "rake"
require "fileutils"
require "yaml"
require "deep_merge"
require "cloudstack_client"
require "time"

template_dir = "templates"

inventory_file = "#{template_dir}/inventories/default.yml"
inventory = YAML.load_file(inventory_file)
config_yml = "config.yml"
raise "#{config_yml} cannot be found" unless File.exist?(config_yml)
config = YAML.load_file(config_yml)
if File.exist?("#{config_yml}.local")
  config.deep_merge!(
    YAML.load_file("#{config_yml}.local")
  )
end

def rake(path:, args:)
  puts "Running rake #{args}"
  Bundler.with_clean_env do
    Dir.chdir(path) do
      sh "bundle exec rake #{args}"
    end
  end
end

def all_targets_of(target, inventory)
  inventory["all"]["hosts"].keys.sort.map { |i| "#{i}:#{target}" }
end

inventory["all"]["hosts"].each_key do |name|
  unless inventory["all"]["hosts"][name].key?("box_version")
    raise "`box_version` for `#{name}` box is not defined in inventory file"
  end
  box_version = inventory["all"]["hosts"][name]["box_version"]
  namespace name do
    output_dir = "output-#{name}"
    desc "initialize build directory"
    task :setup do |_t|
      puts "Creating `#{output_dir}`"
      if Dir.exist?(output_dir)
        puts "`#{output_dir}` already exists, skipping"
      else
        Dir.mkdir(output_dir)
      end
      puts "Copying template to `#{output_dir}`"
      FileUtils.cp_r("#{template_dir}/.", output_dir)
      rake path: output_dir, args: "setup"
    end

    desc "do_build #{name}"
    task :do_build do |_t|
      rake path: output_dir, args: "#{name}:build"
    end

    desc "Create OVA file"
    task :ova do |_t|
      rake path: output_dir, args: "#{name}:ova"
      src = "#{output_dir}/ova/#{name}.ova"
      dst = "#{name}-#{box_version}.ova"
      puts "Copying #{src} to #{dst}"
      FileUtils.cp src, dst
    end

    desc "clean #{output_dir}"
    task :clean do |_t|
      rake path: output_dir, args: "clean"
      puts "Removing `#{output_dir}`"
      FileUtils.rm_rf [output_dir], secure: true
    end

    namespace "upload" do
      desc "uplaod to AWS S3"
      task :s3 do |_t|
        require "aws-sdk-s3"
        file = "#{name}-#{box_version}.ova"
        %w[
          AWS_ACCESS_KEY_ID
          AWS_SECRET_ACCESS_KEY
        ].each do |i|
          raise "environment variable `#{i}` must be set" unless ENV[i]
        end
        unless config.key?("upload") && config["upload"].key?("s3")
          raise "config section for S3 cannot be found"
        end
        raise "`#{file}` cannot be found" unless File.exist?(file)

        s3_config = {
          "key_prefix" => ""
        }.merge(config["upload"]["s3"])
        %w[bucket region].each do |i|
          unless s3_config.key?(i)
            raise "config for S3 does not have a required key: `#{i}`"
          end
        end

        key = "#{s3_config['key_prefix']}#{file}"
        acl = s3_config["acl"]
        bucket = s3_config["bucket"]
        client = Aws::S3::Client.new(region: s3_config["region"])
        puts "uploading to S3"
        puts "bucket: #{bucket}"
        puts "key: #{key}"
        puts "acl: #{acl}"
        r = client.put_object(
          acl: acl,
          body: File.read(file),
          bucket: bucket,
          key: key
        )
        puts "upload #{file} completed."
        puts format("URL: https://s3-%s.amazonaws.com/%s/%s%s",
                    s3_config["region"], s3_config["bucket"],
                    s3_config["key_prefix"], file)
        puts "ETAG: #{r.etag}"
      end
    end

    desc "create VM template"
    task :template do |_t|
      %w[
        CS_KEY_PAIR
        CS_API_KEY
        CS_SECRET_KEY
      ].each do |i|
        raise "environment variable `#{i}` must be set" unless ENV[i]
      end
      unless config.key?("template")
        raise "config section for VM template cannot be found"
      end
      unless config["template"].key?("upload")
        raise "key `upload` is not defined"
      end
      unless config["template"]["upload"] == "s3"
        raise "upload `#{config['template']['upload']}` is not supported"
      end

      unless inventory["all"]["hosts"][name].key?("ostypeid")
        raise "`ostypeid` is not defined in inventory file"
      end
      ostypeid = inventory["all"]["hosts"][name]["ostypeid"]

      unless config["template"].key?("zoneid")
        raise format("`zoneid` is not defined under `template` in %s",
                     config_yml)
      end
      zoneid = config["template"]["zoneid"]
      unless config["template"].key?("cloudstack_api_url")
        raise format(
          "`cloudstack_api_url` is not defined under `template` in %s",
          config_yml
        )
      end

      url = ""
      case config["template"]["upload"]
      when "s3"
        url = format("https://s3-%s.amazonaws.com/%s/%s%s-%s.ova",
                     config["upload"]["s3"]["region"],
                     config["upload"]["s3"]["bucket"],
                     config["upload"]["s3"]["key_prefix"],
                     name,
                     box_version)
      else
        raise format("unsupported uplaod method: %s",
                     config["template"]["upload"])
      end

      cs = CloudstackClient::Client.new(
        config["template"]["cloudstack_api_url"],
        ENV["CS_API_KEY"],
        ENV["CS_SECRET_KEY"]
      )
      puts "creating a template for #{name}"
      cs.register_template(
        format: "OVA",
        hypervisor: "VMware",
        sshkeyenabled: true,
        isextractable: true,
        passwordenabled: true,
        name: format("%s-%s", name, box_version),
        displaytext: format("%s-%s %s", name, box_version, Time.now.iso8601),
        ostypeid: ostypeid,
        url: url,
        zoneid: zoneid
      )
    end

    desc "build #{name} and clean `#{output_dir}`"
    task build: [:setup, :do_build, :ova, :clean]
  end
end

desc "build all"
task build: all_targets_of("build", inventory) do |_t|
end

namespace "upload" do
  desc "upload all OVA files to S3"
  task s3: all_targets_of("upload:s3", inventory) do |_t|
  end
end

desc "create all templates"
task template: all_targets_of("template", inventory) do |_t|
end

desc "clean everything"
task :clean do |_t|
  begin
    Dir.glob("output-*").each do |dir|
      rake path: dir, args: "clean"
    end
  ensure
    puts "Removing `#{Dir.glob('output-*')}`"
    FileUtils.rm_rf Dir.glob("output-*"), secure: true
    puts "Removing `#{Dir.glob('*.ova')}`"
    FileUtils.rm_rf Dir.glob("*.ova"), secure: true
  end
end

namespace :clean do
  desc "Remove all OVA files on S3"

  # XXX sanity check is not DRY
  task :s3 do
    require "aws-sdk-s3"

    objects = []
    s3_config = {
      "key_prefix" => ""
    }.merge(config["upload"]["s3"])
    %w[bucket region].each do |i|
      unless s3_config.key?(i)
        raise "config for S3 does not have a required key: `#{i}`"
      end
    end
    inventory["all"]["hosts"].each_key do |name|
      unless inventory["all"]["hosts"][name].key?("box_version")
        raise "`box_version` for `#{name}` box is not defined in inventory file"
      end
      box_version = inventory["all"]["hosts"][name]["box_version"]
      file = "#{name}-#{box_version}.ova"
      %w[
        AWS_ACCESS_KEY_ID
        AWS_SECRET_ACCESS_KEY
      ].each do |i|
        raise "environment variable `#{i}` must be set" unless ENV[i]
      end
      unless config.key?("upload") && config["upload"].key?("s3")
        raise "config section for S3 cannot be found"
      end

      key = "#{s3_config['key_prefix']}#{file}"
      objects << { key: key }
    end

    bucket = s3_config["bucket"]
    puts "Following objects will be removed from bucket `#{bucket}`:"
    objects.each do |o|
      puts o[:key]
    end
    client = Aws::S3::Client.new(region: s3_config["region"])
    r = client.delete_objects({
      bucket: bucket,
      delete: {
        objects: objects,
        quiet: false
      }
    })
    puts "Deleted objects:"
    r.deleted.each do |o|
      puts "key: #{o.key}"
    end
    r = client.list_objects({
      bucket: bucket,
      prefix: s3_config["key_prefix"]
    })
    puts "Remaining objects with prefix `#{s3_config["key_prefix"]}` in bucket `#{bucket}`:"
    r.contents.each do |o|
      puts o.key
    end
  end
end
