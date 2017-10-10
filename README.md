# `box2cloudstack`

Creates `.ova` file for `cloudstack` from `vagrant` box.

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
