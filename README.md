# vagrant-fess 

[Fess](http://fess.codelibs.org/) (Full tExt Search System) is an Open Source Enterprise Search Server.

Use this Vagrantfile and CloudInit configuration to install Fess, Elasticsearch and Haproxy on a CentOS-7 VM on VirtualBox.

## Usage Instructions

### Prerequisites

- [VirtualBox](https://www.virtualbox.org) (5.1.16)
- [Vagrant](https://www.vagrantup.com) (1.9.2)

### Installation

Copy `config.rb.example` to `config.rb` and apply any necessary configuration settings. Increasing both `$vm_memory` and `$vm_cpus` is recommended.

```
$ vagrant up
```

### Usage

After installation and initialisation has completed you will receive a message indicating the URL FESS is available on.

For usage instructions refer to the [Fess Documentation](http://fess.codelibs.org/).

## Known Issues

Currently, the General Configuration option Thumbnail View is not supported. [PhantomJS](http://phantomjs.org/) is required to support this feature.

## Credits

- [CodeLibs Project](https://github.com/codelibs) - Maintainers of the Fess Project.
