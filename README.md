Table of Contents
=================

  * [Table of Contents](#table-of-contents)
  * [box2cloudstack](#box2cloudstack)
    * [Requirements](#requirements)
    * [Usage](#usage)
    * [config.yml](#configyml)
      * [template top level key](#template-top-level-key)
      * [upload top level key](#upload-top-level-key)
        * [s3](#s3)
    * [Targets](#targets)
      * [build](#build)
      * [upload:s3](#uploads3)
      * [clean](#clean)
    * [Creating a VM from templates](#creating-a-vm-from-templates)

# `box2cloudstack`

Creates `.ova` file for `cloudstack` from `vagrant` box.

## Requirements

* ansible
* VirtualBox
* vagrant
* bundler
* [ovftool](https://www.vmware.com/support/developer/ovf/) (registration required)

Either one of:

* [cloudstack-api](https://github.com/idcf/cloudstack-api)
* [cloudstack-cli](https://github.com/niwo/cloudstack-cli) (XXX not supported yet)

## Usage

```
bundle install --path vendor/bundle
```

After that, build all boxes:

```
bundle exec rake build
```

Or, build a box

```
bundle exec rake ansible-freebsd-10.3-amd64:build
```

`Rakefile` creates `OVA` file in the top directory

```
ls *.ova
```

To build a box again, run

```
bundle exec rake ansible-freebsd-10.3-amd64:clean
```

The above command removes running VMs and the work directory (but NOT created
`OVA` file).

To clean everything including `OVA` files, run:

```
bundle exec rake clean
```

After successful build, upload the OVA file. Currently AWS S3 is supported.

```
bundle exec rake ansible-freebsd-10.3-amd64:upload:s3
```

Lastly, create a template.

```
bundle exec rake ansible-freebsd-10.3-amd64:template
```

The following command will:

* build everything
* upload all OVA files
* create all templates
* clean everything

```
bundle exec rake build upload:s3 template clean
```

## `config.yml`

`config.yml` is a configuration file for the build.

If `config.yml.local` exists, the content of it overrides `config.yml`.

### `template` top level key

This key specifies configurations for registering templates.

A list of currently supported API command is:

* `cloudstack-api`

| Key | Description | Mandatory? |
|-----|-------------|------------|
| `upload` | the name of upload method | yes |
| `zoneid` | the zone ID where the template is registered in | yes |
| `api_command` | command name to execute | yes |

```yaml
template:
  upload: s3
  zoneid: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
  api_command: cloudstack-api
```

### `upload` top level key

This key specifies configurations for each supported upload method.

#### `s3`

To upload OVA files to S3, you need:

* to create a writable bucket
* to permit the cloudstack system to access the bucket

| Key | Description | Mandatory? |
|-----|-------------|------------|
| `region` | region name | yes |
| `bucket` | name of bucket | yes |
| `key_prefix` | prefix of key, i.e. when box name is `ansible-freebsd-10.3-amd64`, and the `key_prefix` is `foo/bar/`, the key of the object is `foo/bar/ansible-freebsd-10.3-amd64`. | no |
| `acl` | ACL string for the object | one of ACL strings supported by S3. see [Access Control List (ACL) Overview](http://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html) | yes |

```yaml
upload:
  s3:
    region: ap-northeast-1
    bucket: mybucket
    key_prefix: ova/cloudstack/
    acl: authenticated-read
```

## Targets

### build

This target builds all OVA files defined in `templates/inventories/default.yml`.

### upload:s3

This uploads all OVA files to AWS S3.

### clean

Clean everything, including OVA files.

## Creating a VM from templates

See [examples/Vagrantfile](examples/Vagrantfile).

You need to:

* have SSH key registered (choose key name to match path to `~/.ssh/$CS_KEY_PAIR`)
* obtain API key and secret
* export SSH key name, API key, API secret into environment variables

```sh
export CS_API_KEY=YOUR_KEY
export CS_SECRET_KEY=YOUR_SECRET_KEY
export CS_KEY_PAIR=YOUR_SSH_KEY_NAME
vi Vagrantfile
vagrant up
```

## Attaching additional disk

`vagrant-cloudstack` does not implement `size` in `deployVirtualMachine` API,
which is required to create and attach a custom volume.

For `v1.4.0`, you need to patch the gem.
[v1.4.0-size](https://github.com/trombik/vagrant-cloudstack/tree/v1.4.0-size)
is the patch. Build and install the gem.

Or, you may install [pre-built gem](files/vagrant-cloudstack-1.4.0.gem).

```
vagrant plugin install files/vagrant-cloudstack-1.4.0.gem
```
