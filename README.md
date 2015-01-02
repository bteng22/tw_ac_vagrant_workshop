Chef Tutorial
=============


Prerequisites
----------------------------  

install Sublime from http://www.sublimetext.com/3 (optional)  
install vagrant from https://www.vagrantup.com/downloads  
install virtualbox from https://www.virtualbox.org/wiki/Downloads  
install chef-dk from https://downloads.chef.io/chef-dk/mac/#/  

Let's verify that these instalations were successful. The following command should return 1.7.1 or greater
```
$ vagrant --version 
````

The following command should open up the virtualbox manager application 
```
$ virtualbox
````  

The chef-dk includes chef, test-kitchen, and berkshelf. To verify that all of these were installed properly run
```
$ chef --version 
$ kitchen --version 
$ berks --version
```

Cloning the repo
----------------

First clone the workshop repo    
`$ git clone git@github.com:ekcasey/tw_ac_vagrant_workshop.git`    

This repo contains two directories. The 'app' directory includes a ruby app called Minions. Our goal is to use Chef to provision a virtual machine machine so that it can run the Minions app. Take a minute to inspect the app directory.

The cookbook directory will hold our chef code. We have created the basic directory structure for you. Take a moment to look at the metadata.rb. The following lines describe the name and version of the cookbook.

```
name    'wdiy'
version '0.0.1'
```

Later we will be specifying our cookbook dependencies in this metadata file.


Setting up the Local Environment
----------------------------

### Setting Up Berkshelf  

Berkshelf is a dependency manager for Chef. The cookbook we write will depend on cookbooks written by the chef community. You wil notice that an empty Berksfile has been created for you. Berkshelf uses the Berksfile to determine what dependencies to fetch and where to fetch these dependencies from. Add the following lines to your Berksfile.  

Berksfile

```
source 'https://supermarket.getchef.com'

metadata
```

The first line tells Berkshelf to fetch cookbooks from the chef supermarket. The second line tells Berkshelf to inspect the cookbook's metadata file to determine dependencies (don't worry we will try this out later). To verify that your Berksfile is set up correctly, cd into the cookbook directory and make sure the following command executes without errors (nothing will be downloaded because we haven't specified any dependencies yet).  

`$ berks install`  

###  Setting Up Test Kitchen

To borrow directly from the test-kitchen project 'Test Kitchen is an integration tool for developing and testing infrastructure code and software on isolated target platforms'. What this means is that test-kitchen is the glue that holds our toolchain together. The kitchen command line tool allows you to create virtual machines using vagrant or another driver, upload your cookbook and its dependencies to that machine using berkshelf, apply your cookbook using chef, and then verify the state of the machine using minitest or another testing framework (we won't actaully try testing today). We have already configured test-kitchen for this project, But lets take a moment to look at the key pieces of the .kitchen.yml file...  
  
```
driver:
  name: vagrant
```
We have specified that we will use vagrant (on top of virtualbox) as a driver. This means that kitchen will create instances using vagrant (other drivers allow you to create instances in the cloud using various cloud drivers).  

```
provisioner:
  name: chef_solo
```

We have specified that we will be using chef_solo as a provisioner (this means that we will not be needing a chef server)

```
platforms:
  - name: centos-6.4
    driver:
      box: centos65-x86_64-20140116
```
The above lines specifies the type of instance kitchen should create. We will be testing our cookbook on  centos-6.4 image. The last 2 lines tell test-kitchen which base image to use when creating instances. We need to download this image so that it is available to test-kitchen   

Lets download the base box...  this will take about 3 min...

```
$ vagrant box add centos65-x86_64-20140116 https://github.com/2creatives/vagrant-centos/releases/download/v6.5.3/centos65-x86_64-20140116.box --insecure
```

Finally lets take a look at the suites section of the .kitchen.yml 
```
suites:
  - name: wdiy
```

Here we have defined one suite named wdiy.

The first important test-kitchen command you should know is `kitchen list`. This prints out the instances that kitchen knows about. When you run `kitchen list` it should list a single instances named 'wdiy-centos-64' that is currently '<Not Created>'

The next important command is `kitchen create`. This will create that instance but will not yet apply the cookbook. Let's try running it now...
 
`$ kitchen create`  

This command should execute succesfully and create your virtual machine. To verify that the virtual machine was successfully create open virtual manager and look for an instance with the name  'wdiy-centos-64'.

Next, we want to ssh into our newly created virtual machine. To do this run the following command
```
$ kitchen login
```  

Now you are logged into the virtual machine! Your command prompt should have changed to indicate this. From now on we will call this machine the VM or th guest. Take a moment to look around. Maybe try the following commands...  
```
$ whoami  
$ hostname    
$ ip addr  
```  

now lets return to the host machine  
`$ exit`  

If we run `kitchen list` now, we should see that the status of our instance is 'Created'.
 

Your First Cookbook
-------------------

**Main Objective**: Write a Chef cookbook that will provision an application server for the Minions app.

### Sharing a Folder
How will we get the app code onto the virtual machine? For now, lets share a directory from our host machine with the guest VM. Add the following line to your drvier configuration in .kitchen.yml.
```
  synced_folders:
    - ["../app", "/minions"]
```  

Since we have changed our VM configuration we must destroy the VM and recreate it.

```
$ kitchen destroy
$ kitchen create
```

Now lets test whether the synced folder is working    
`$ kitchen login`    
`$ cd /`  
`$ ls`  

You should see the minions directory on the virtual machine  

```
$ cd minions   
$ ls
```

### Install Ruby With Chef

Since this is a ruby app we must install ruby on the guest box in order to run the Minions app. 

We are going to use the rbenv cookbook to install ruby. First we need to add the rbenv cookbook as a dependency in our metadata file. Add the following line to metadata.rb

```
depends 'rbenv', '1.7.1'
```

Take a moment to look at the rbenv cookbook documentation https://supermarket.chef.io/cookbooks/rbenv/versions/1.4.1

Now we can use the rbenv_ruby resource to install ruby globally on the vagrant machine. First let's create a defualt.rb file in the recipes directory

`touch recipes/default.rb`

Now add the following lines to this file

```
include_recipe "rbenv::default"
include_recipe "rbenv::ruby_build"
```

Now all of the resource types defined in the rbenv cookbook are available to us. Lets use the rbenv_ruby resource. Add the following lines to your default.rb

```
rbenv_ruby '2.1.1' do
  global true
end
```

Now we must add our cookbook to the vagrant runlist. Add the following attribute to the wdiy suite in .kitchen.yml
```
    run_list:
      - wdiy
```

Now lets apply the cookbook to the instance. 'kitchen converge' will apply your run_list to a created instance.
```
$ kitchen converge
```


Run `$ kitchen login` to ssh into the box.  
On the vagrant machine run `ruby --version`. The command should print 2.1.1 to the console.  

Now, that we have ruby installed lets try running the app.

```
cd /minions/lib
ruby run_app.r
```
Oops! You should see the following error 'cannot load such file -- sinatra (LoadError)'. We need to install bundler on the VM so that we can install Minion's dependencies. Your turn!

*Excercise 1: Install Bundler*  
*Extend default.rb so that it installs the bundler gem on the guesst. Hint: look at the rbenv docs. When you are ready, coverge your instance. Now login, and try running `bundle install` in the minions directory. Were you successful? If not, keep trying!*

When you have successfully installed Minion's dependencies try starting the app again. If it starts successfully you should see the following message 'Sinatra/1.4.5 has taken the stage on 4567 for development with backup from WEBrick'. Lets quickly verify this with curl. The following command should return 'Hello Minions!'
```
$ curl http://localhost:4567
``` 

Next, we would like to see 'Hello Minions!' displayed in a browser. This is a little trickier b/c our guest machine has no browser. We want to use the browser on our host machine. To do this we woudl like to forward port 4567 from the guest to the host machine. Therefore when we access localhost:4567 in our host browser it will display content from the guest at port 4567. 

To set up the forwarded port, add the following line to your driver configuration in .kitchen.yml.

```
  network:
    - ["forwarded_port", {guest: 4567, host: 4567}]
```  

Now 



Start application. You should see a database error.

Install Mysql With Chef

Create Minion Database  

*Excercise 2: Extract the ruby version into an attribute*  
*Rembember what we learned about attributes? Extract the ruby version into an attribute so that it is configurable. Make sure to test your work.*  








