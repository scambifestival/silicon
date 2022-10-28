## web server for social.scambi.org

**procedure**

follow Debian 11 template  

swap tuning  
>nano /etc/sysctl.d/88-tuning.conf

    vm.swappiness = 1
    vm.vfs_cache_pressure = 150

>sysctl --system

install useful packages  
>apt install screen git rsync curl wget gnupg

nodejs repo
>curl -fsSL https://deb.nodesource.com/setup_lts.x | bash -  

prerequirements installation  
>apt install imagemagick ffmpeg libpq-dev libxml2-dev libxslt1-dev file git-core g++ libprotobuf-dev protobuf-compiler pkg-config nodejs gcc autoconf bison build-essential libssl-dev libyaml-dev libreadline6-dev zlib1g-dev libncurses5-dev libffi-dev libgdbm-dev nginx redis-server redis-tools postgresql postgresql-contrib certbot python3-certbot-nginx libidn11-dev libicu-dev libjemalloc-dev msmtp-mta  

msmtp configuration  
>nano /etc/msmtprc

    defaults
    auth on
    tls on  

    account gandi
    host mail.gandi.net
    port 587
    tls_starttls on
    from staff@scambi.org
    user staff@scambi.org
    password abcdef   

    account default : gandi

>systemctl enable --now msmtpd

yarn configuration
>corepack enable  
>yarn set version stable  

postgresql configuration
>/etc/postgresql/13/main/conf.d/88-tuning.conf  

    max_connections = 25
    shared_buffers = 256MB
    effective_cache_size = 768MB
    maintenance_work_mem = 64MB
    checkpoint_completion_target = 0.9
    wal_buffers = 7864kB
    default_statistics_target = 100
    random_page_cost = 4
    effective_io_concurrency = 2
    work_mem = 10485kB
    min_wal_size = 1GB
    max_wal_size = 4GB
    max_worker_processes = 2
    max_parallel_workers_per_gather = 1
    max_parallel_workers = 2
    max_parallel_maintenance_workers = 1

>systemctl restart postgresql

>sudo -Hiu postgres psql  
>>CREATE USER mastodon CREATEDB;  
>>\q  

ruby installation
>adduser --disabled-login mastodon  
>su - mastodon  

>git clone https://github.com/rbenv/rbenv.git ~/.rbenv  
>cd ~/.rbenv && src/configure && make -C src  
>echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc  
>echo 'eval "$(rbenv init -)"' >> ~/.bashrc  
>exec bash  

>git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build  
>RUBY_CONFIGURE_OPTS=--with-jemalloc rbenv install 3.0.3  
>rbenv global 3.0.3  
>gem install bundler --no-document  

>cd  
>git clone https://github.com/tootsuite/mastodon.git live  
>cd live  
>git checkout $(git tag -l | grep -v 'rc[0-9]*$' | sort -V | tail -n 1)  

>bundle config deployment 'true'  
>bundle config without 'development test'  
>bundle install -j$(getconf _NPROCESSORS_ONLN)  
>yarn install --pure-lockfile  

>RAILS_ENV=production bundle exec rake mastodon:setup  
>>Domain name: **social.scambi.org**  
>>Do you want to enable single user mode? (y/N) **n**  
>>Are you using Docker to run Mastodon? (Y/n) **n**  
>>*press enter for postgresql and redis options*  
>>Do you want to store uploaded files on the cloud? (y/N) **n**  
>>Do you want to send e-mails from localhost? (y/N) **y**  
>>E-mail address to send e-mails "from": (Mastodon \<notifications@social.scambi.org\>) **Pan \<staff@scambi.org\>**  
>>Send a test e-mail with this configuration right now? **n**  
>>Save configuration? (Y/n) **y**  
>>Prepare the database now? (Y/n) **y**  
>>Compile the assets now? (Y/n) **y**  
>>Do you want to create an admin user straight away? (Y/n) **y**  
>>Username: **silicon**  
>>E-mail: **silicon@scambi.org**  
>>*write the password in Keepass*  

>exit  

firewall configuration  
>firewall-cmd --permanent --zone=public --add-service={http,https}  
>firewall-cmd --reload

nginx configuration  
>nano /etc/nginx/nginx.conf

    server_tokens off;

>cp /home/mastodon/live/dist/nginx.conf /etc/nginx/sites-available/social  
>sed -i 's/example.com/social.scambi.org/g' /etc/nginx/sites-available/social  
>rm /etc/nginx/sites-enabled/default  
>systemctl restart nginx  

>certbot --nginx -d social.scambi.org  

uncomment certificate parameters in the file /etc/nginx/sites-available/social  

>ln -s /etc/nginx/sites-available/social /etc/nginx/sites-enabled/social  
>systemctl restart nginx  

>cp /home/mastodon/live/dist/mastodon-*.service /etc/systemd/system/  
>systemctl daemon-reload  
>systemctl enable --now mastodon-web mastodon-sidekiq mastodon-streaming  

finish the configuration on https://social.scambi.org

fix crontab
>crontab -e

    0 4 * * * RAILS_ENV=production /home/mastodon/live/bin/tootctl media remove
    0 5 * * * RAILS_ENV=production /home/mastodon/live/bin/tootctl preview_cards remove


**local backup**

**remote backup**
