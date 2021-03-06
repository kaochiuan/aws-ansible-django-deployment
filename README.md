Deploying Django on AWS (Ubuntu)
======

I've put this repository together with the intention that it can be used as a teach yourself tutorial for deploying Django with Ansible.  Have fun deploying and please feel free to contribute if you think it can be improved.

The workflow is as follows:

1. Spin up an EC2 Ubuntu server via AWS or [Vagrant](http://www.vagrantup.com/) - a sample Vagrant file is included
2. Provison the server using [Ansible](http://www.ansible.com/home)
3. Choose either RDS from Amazon, PostgreSQL or MYSQL database
4. Deploy your Django project from Git and serve it with this stack:
  * [Gunicorn](http://gunicorn.org/)
  * [Nginx](http://nginx.org/)
  * [Supervisord](http://supervisord.org/)
  * [Memcached](http://memcached.org/)
  * [Virtualenv](http://virtualenv.readthedocs.org/en/latest/)
5. Optionally install nodejs, mongodb, elastic search or celery with reddis

This stack comes with useful logging for gunicorn, supervisord, and nginx.  It uses logrotate for managing logs and [aws-snapshot-tool](https://github.com/evannuil/aws-snapshot-tool) for rudimentary (full server image) backups.  You can also use this playbook to deploy multiple apps to the same server.

`*** You probably don't want to use this for live use, it's more about providing a quick dev enviroment for common stack components ***`


##2 minute quick start

Start by opening the `playbook.yml` file and providing values for all the non-commented variables listed under `vars`.  The application will be deployed and owned by `application_user`.  You'll need to provide their password hash by running the following command:

`python -c "from passlib.hash import sha512_crypt; print sha512_crypt.encrypt('your-password')"`

Later if you log into the server, you can easily change to this user with `su - [application_user]`.  When you do this you'll notice the venv is auto-activated and you are automatically in your web app directory.

Finally, open the `hosts` file and append your server IP underneath `[webservers]`.

Now, run the playbook against your server with the following command:

`ansible-playbook playbook.yml -i hosts --private-key=/Path/to/AWS/key/your.pem`

Should any task fail, you can make corrections and run again from that task:

`ansible-playbook playbook.yml -i hosts  --start-at-task='my task name'`


#Assumptions

This project assumes you have a `live_settings.py` in additon to a `settings.py` and that your `application_name` variable is named the same as your Django project.  You'll also need to edit your `wsgi.py` file to point to `live_settings.py` rather than the default `settings.py`.

It's also assumed your project is set up as below.  Other variations are of course possible but you'll have to change the `virtualenv_path`, `git_root`, and `django_dir` variables accordingly.

```
./media
./static
./requirements.txt
./application_name
		./app1
		./app2
		manage.py
		./application_name
			settings.py
			live_settings.py
			wsgi.py
			...
```

Finally, all sensitive settings are enviroment variables which are exported via the postactivate script (in roles/web/templates).  The example below, from [2 scoops of Django](http://twoscoopspress.org/products/two-scoops-of-django-1-6) is a nice example of how you can configure your local/testing/live settings files.

```
def get_env_variable(var_name):
  """ Get the environment variable or return exception """
  try:
    return os.environ[var_name]
  except KeyError:
    error_msg = "Set the %s environment variable" % var_name
    raise ImproperlyConfigured(error_msg)

DATABASES = {
  'default': {
    'ENGINE': 'django.db.backends.postgresql_psycopg2',
    'NAME': get_env_variable('DB_NAME'),
    'USER': get_env_variable('DB_USER'),
    'PASSWORD': get_env_variable('DB_PASSWORD'),
    'HOST': get_env_variable('DB_HOST'), # Empty for localhost through domain sockets or '127.0.0.1' for localhost through TCP.
    'PORT': '', # Set to empty string for default.
},
SECRET_KEY = get_env_variable('DJANGO_SECRET_KEY')
```


##SSH agent forwarding and cloning your Git repository

You might have some issues with Ansible timing out or giving a public key denied errors when trying to connect to your private repository.  [This guide](https://help.github.com/articles/using-ssh-agent-forwarding) is pretty useful for troubleshooting these errors.


##Variable management and Ansible Vault

Ansible Vault allows you to easily encrypt .yml files which you'll want to commit version control.  The vars directory is a good place to organise your encrypted variable files.

`ansible-vault encrypt foo.yml bar.yml baz.yml #to encrypt existing files`

`ansible-vault create foo.yml #to create a new encrypted file`

`ansible-vault edit foo.yml`

To run a playbook with encrypted files:

`ansible-playbook site.yml --ask-vault-pass`


##Ongoing deployments

Use tags to run a bespoke collection of tasks on your hosts

`ansible-playbook build/playbook.yml -i build/inventory/hosts  --tags='deploy'`

Some useful server commands:

`sudo supervisorctl restart application_name`

Restart nginx, postgres, etc

`sudo service restart nginx`


##AWS credentials

If you want to use Vagrant to spin up your AWS servers you'll need to get the following details from your AWS account.

  * ACCESS_KEY_ID
  * SECRET_ACCESS_KEY

Next, if you haven't already, you'll need to generate a keypair and configure the security groups and subnets for your VPC.  Having done all this, you should go to the file called 'local_envs' in this directory and fill in all your information.  The database variables in this file are intended for your local database installation.

Finally, install [Vagrant](https://docs.vagrantup.com/v2/installation/) and this [AWS provider](https://github.com/mitchellh/vagrant-aws) for Vagrant

At this stage you should be able to create an EC2 instance via Vagrant.

To verify, source your variables in the 'envs' file and run the following:

`vagrant up --provider=aws`

`vagrant ssh` to take a look around, then `exit` to leave

Back on your host you can checkout your server details with:
    `vagrant ssh-config`

And finally clean up with:
    `vagrant destroy`

Should Vagrant fail to complete your provisioning, you can always just run Ansible picking up from where you left off.


##Vagrant
Once you've configured your enviroment and Ansible variables and you've followed the step above you'll be able to both spin up and provision by simply typing:

`vagrant up --provider=aws`

Thereafter, you'll be able to make ongoing deployments and updates using Ansible.


##To do
Dynamic inventories, Development, Staging, Live setups, Better lockdown etc


##Credits
I've mainly borrowed from or been influenced by this [repo](https://github.com/jcalazan/ansible-django-stack) which was inspired by this excellent [blogpost](http://michal.karzynski.pl/blog/2013/06/09/django-nginx-gunicorn-virtualenv-supervisor/).