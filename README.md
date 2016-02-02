# Some ChefDK Best Practices

## Synopsis of Problem
Sometimes, you may find that you require some extra functionality that isn't included in the ChefDK bundle. Generally the first place to look is the addition of 3rd party Gems. Without a good documented strategy on how to best accomplish this, things can go sideways pretty quickly.

Knowing that your goal should be to be able to wipe your workspace clean of ChefDK and 3rd Party Gems with each upgrade and have them re-installed in a deterministic, repeatable and idempotent fashion, the following are some suggestions.

## Uninstalling Previous ChefDK Versions
First, the best way to keep your workspace clean and predictable is to not let old artifacts stick around that can pollute and add variant behavior quickly.  This starts with a proper strategy for removing old versions of ChefDK and other Gems:

You can find your OS uninstall method here:
https://docs.chef.io/install_dk.html#uninstall

NOTE: Additionally, since 3rd Party Gems will be installed into `~/.chefdk` you should also remove those items.

```rm -rf ~/.chefdk/gem```

These gems will be re-installed later (see 3rd Party Gems below)

## Adding 3rd Party Gems into the Mix
It is important to stress that adding 3rd Party Gems adds complexity and opportunity for introducing maintenance overhead.  If you can, by all means, keep to just a pure ChefDK and nothing more.

However, if you must add additional Gems, it is important to understand where things go with ChefDK. `chef gem env` gives you a clear picture.

```
...
  - GEM PATHS:
     - /opt/chefdk/embedded/lib/ruby/gems/2.1.0
     - /Users/jmiller/.chefdk/gem/ruby/2.1.0
...

```  

All Gems included in ChefDK installer package are placed into the `/opt/chefdk/embedded/lib/ruby/gems` or `C:/opscode/chefdk/embedded/lib/ruby/gems/` location.  All additional gems that you add will be installed into `~/.chefdk/gem` or `C:/Users/<user>/AppData/Local/chefdk/gem/` by default.  If you attempt to install them in any other locations then the ones above in GEM_PATHS, you will have to manipulate your GEM_PATH and other settings for the ChefDK components to pick them up; thus it is best to keep everything in the default `~/.chefdk/gem`

So how do we best add additional Gems?

## Good Solution

In order to install Gems into `~/.chefdk/gem` one could use the `chef gem install <gem>` command.  However, the problem with this approach is that you must remember to re-install them manually every time you upgrade the ChefDK (since you're going to be removing that directory) and also, you aren't utilizing a policy based installation; it's outside of version control and it's manual.

The best approach would be to utilize a policy based mechanism that exists in source control so that it can be reviewed, is easily apparent and is repeatable.  One example: a Cookook for installing ChefDK that uses 'chef_gem' or 'gem_package'

## Worse Solution
Using Bundler (see: http://bundler.io/v1.5/rationale.html) On the surface bundler seems like a great way to manage gem installs for ChefDK environments.  However, the result is divergent from what we want:

<b>To introduce as little change as possible to a pure ChefDK configuration.</b>

To see what I mean I will illustrate an example.  Let's say ChefDK provides everything I need, with the exception of the 'kitchen-ec2' gem which I need for integration testing.  So I create the simplest Gemfile possible:

```
source 'https://rubygems.org'

group :testing do
  gem "kitchen-ec2", "= 0.10.0"
end
```

After a `bundle install` I end up with 10 additional gems in `~/.chefdk/gem` including NEWER versions of gems that are ALREADY bundled in my ChefDK install (ie. test-kitchen and mixlib-shellout)

To take this example further, let's say I have a Rakefile that I utilize to test my cookbook:

```
require 'rspec/core/rake_task'
require 'rubocop/rake_task'
require 'foodcritic'
require 'kitchen/rake_tasks'

...
namespace :test do

  desc 'Run ChefSpec tests'
  RSpec::Core::RakeTask.new(:unit)

  desc 'Run Test Kitchen on AWS EC2'
  task :converge_ec2  do
    Kitchen.logger = Kitchen.default_file_logger
    @loader = Kitchen::Loader::YAML.new(local_config: './.kitchen.ec2.yml')
    config = Kitchen::Config.new(loader: @loader)
    config.instances.each do |instance|
      puts "Testing Instance #{instance.name}"
      instance.test(:always)
    end
  end
end
```

Now, when I run it: `chef exec rake test:unit` The Gem Libraries that are in my `~/.chefdk/gem` path will be loaded FIRST!  I'm utilizing NEWER libraries then the ones that were tested and certified to work with the shipped version of ChefDK I installed!  This will undoubtedly cause variant behavior and headaches.

If you wonder why things like the `kitchen` and `chef` ChefDK commands continue to work without issue, take a look at contents of those files:

```
# file: /opt/chefdk/bin/kitchen
#!/opt/chefdk/embedded/bin/ruby
#--APP_BUNDLER_BINSTUB_FORMAT_VERSION=1--
ENV["GEM_HOME"] = ENV["GEM_PATH"] = nil unless ENV["APPBUNDLER_ALLOW_RVM"] == "true"
gem "mixlib-shellout", "= 2.2.3"
gem "net-scp", "= 1.2.1"
gem "net-ssh", "= 2.9.2"
gem "safe_yaml", "= 1.0.4"
gem "thor", "= 0.19.1"
gem "test-kitchen", "= 1.4.2"
...
```

All the gem dependencies are explicitly listed and pinned.  They are therefore utilizing the versions from the default ChefDK path: `opt/chefdk/embedded/lib/ruby/gems`.  To replicate this determinism in your Gemfile, you would have to match and pin every gem dependency from the ChefDK in your cookbook Gemfile - no way!  Even if you choose some default, high-level gems to pin, you are taking on all the overhead responsibility for keeping track of and testing how they function with your installed ChefDK..

## Summary
If you need extra gems in your ChefDK based development or CI environment, don't utilize bundler, it's great for other purposes, not this.  Utilize some other method like a cookbook to install gems.  Do not install Gems that are already included in the ChefDK (ie. kitchen, rubocop, foodcritic, chefspec etc) install only the gem that gives you the extra functionality you are missing from ChefDK.  Wipe out `~/.chefdk/gem` with every new install - they will be re-installed by your policy based method above.
