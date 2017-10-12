# `box2cloudstack`

Creates `.ova` file for `cloudstack` from `vagrant` box.

## Requirements

* ansible
* VirtualBox
* vagrant
* bundler
* [ovftool](https://www.vmware.com/support/developer/ovf/) (registration required)

Optionally, but highly recommended to install cloudstack API CLI
tool. Some candidates:

* [cloudstack-cli](https://github.com/niwo/cloudstack-cli)
* [cloudstack-api](https://github.com/idcf/cloudstack-api)

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
