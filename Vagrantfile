# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'fileutils'
startSSHPort = 2250
vagrant_root = File.dirname(__FILE__)
dev_suffix = ENV['DEV_SUFFIX'] || "local"
dev_domain = ENV['ENJO_DEV_DOMAIN'] || 'enjo.test'
mysql_password = ENV['MYSQL_ROOT_PASSWORD'] || "root"
persistent_storage = vagrant_root + '/persistent_storage'
mode = ENV['VAGRANT_MODE'] || 'dev'
ip_range = ENV['DEV_IP_RANGE'] || "172.23.0"
reverseproxynodes = {
        'www' => '20',
        'sante' => '30'
}
storefrontnodes = {
        'www' => '40',
        'sante' => '50'
}
apinodes = {
        'www' => '60',
        'sante' => '70'
}
puts "========================================================"
puts "domain : #{dev_domain}"
puts "folder : #{vagrant_root}"
puts "mysql root password : #{mysql_password}"
puts "mode: #{mode}"
puts "persistent storage: #{persistent_storage}"
puts "ip range used: #{ip_range}"
puts "dev suffix: #{dev_suffix}"
puts "========================================================"

FileUtils.mkdir_p(persistent_storage)
FileUtils.mkdir_p(persistent_storage+"/mysql")
FileUtils.mkdir_p(persistent_storage+"/elasticsearch")
FileUtils.mkdir_p(persistent_storage+"/composer")
FileUtils.chmod 0777, persistent_storage+"/elasticsearch";
if not File.exist?("#{persistent_storage}/mysql/ibdata1")
  FileUtils.chmod 0777, persistent_storage+"/mysql";
end
FileUtils.chmod 0777, persistent_storage+"/composer";

Vagrant.configure('2') do |config|
    config.vm.boot_timeout = 1800
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.manage_guest = true
    config.hostmanager.ignore_private_ip = false
    config.hostmanager.include_offline = false
    apinodes.each do |name, ip|
        config.vm.define "api-#{name}", primary: false do |vueapi|
            vueapi.hostmanager.aliases = [ "api.#{name}."+dev_domain ]
            vueapi.communicator.bash_shell = '/bin/sh';
            vueapi.trigger.before :all do |trigger|
                # check that the /tmp/vueapi folder exists (which is used to simulated the tmpfs setup as per vue composer files
                if File.directory?("/tmp/vueapi_#{name}")
                    FileUtils.rm_rf("/tmp/vueapi_#{name}")
                    FileUtils.mkdir_p("/tmp/vueapi_#{name}")
                    trigger.info = "Temp folder /tmp/vueapi_#{name} created and will be mapped to the docker filesystem as a volume"
                end
                trigger.ignore = [:destroy, :halt]
            end

             if File.exist?("#{vagrant_root}/vsf_boot.sh")
                    vueapi.vm.provision "shell", path: "#{vagrant_root}/vsf_boot.sh", privileged: true
            end

            vueapi.vm.network :private_network, ip: "#{ip_range}.#{ip}", subnet: "#{ip_range}.0/16"
            vueapi.vm.hostname = "api-#{name}"
            vueapi.vm.communicator = 'docker'
            vueapi.vm.provider 'docker' do |d|
                d.build_dir = "#{vagrant_root}/sites/vue-storefront-api/"
                d.dockerfile = "docker/vue-storefront-api/Dockerfile"
                d.has_ssh = false
                d.name = "api-#{name}"
                d.remains_running = true
                d.volumes = [
                    "#{vagrant_root}/sites/vue-storefront-api/config:/var/www/config",
                    "#{vagrant_root}/sites/vue-storefront-api/ecosystem.json:/var/www/ecosystem.json",
                    "#{vagrant_root}/sites/vue-storefront-api/migrations:/var/www/migrations",
                    "#{vagrant_root}/sites/vue-storefront-api/package.json:/var/www/package.json",
                    "#{vagrant_root}/sites/vue-storefront-api/babel.config.js:/var/www/babel.config.js",
                    "#{vagrant_root}/sites/vue-storefront-api/tsconfig.json:/var/www/tsconfig.json",
                    "#{vagrant_root}/sites/vue-storefront-api/nodemon.json:/var/www/nodemon.json",
                    "#{vagrant_root}/sites/vue-storefront-api/scripts:/var/www/scripts",
                    "#{vagrant_root}/sites/vue-storefront-api/src:/var/www/src",
                    "#{vagrant_root}/sites/vue-storefront-api/var:/var/www/var",
                    "/tmp/vueapi_#{name}:/var/www/dist",
                    "#{vagrant_root}/sites/vue-storefront-api/config/local.json.#{name}.#{dev_suffix}:/var/www/config/local.json"
                    ]
                d.env = { "BIND_HOST" => "0.0.0.0",
                          "ELASTICSEARCH_HOST" => "elasticsearch."+dev_domain,
                          "ELASTICSEARCH_PORT" => "9200",
                          "REDIS_HOST" => "redis."+dev_domain,
                          "VS_ENV" => "#{mode}",
                          "PM2_ARGS" => "--no-daemon",
                          "NODE_TLS_REJECT_UNAUTHORIZED" => "0"
                        }
            end
        end

    end
    storefrontnodes.each do |name, ip|
        config.vm.define "frontend-#{name}", primary: false do |vuestorefront|
            vuestorefront.communicator.bash_shell = '/bin/sh';
            vuestorefront.hostmanager.enabled = true
            vuestorefront.hostmanager.aliases =  [ "frontend-#{name}."+dev_domain ]
            vuestorefront.trigger.before :all do |trigger|
                # check that the /tmp/vuestorefront folder exists (which is used to simulated the tmpfs setup as per vue composer files
                trigger.name = "tmp folder"
                if File.directory?("/tmp/vuestorefront_#{name}")
                    FileUtils.rm_rf("/tmp/vuestorefront_#{name}")
                    FileUtils.mkdir_p("/tmp/vuestorefront_#{name}")
                    trigger.info = "Temp folder /tmp/vuestorefront_#{name} created and will be mapped as a volume into docker node"
                end
                trigger.ignore = [:destroy, :halt]
            end
            if File.exist?("#{vagrant_root}/vsf_boot.sh")
                    vuestorefront.vm.provision "shell", path: "#{vagrant_root}/vsf_boot.sh", privileged: true
            end
            vuestorefront.vm.network :private_network, ip: "#{ip_range}.#{ip}", subnet: "#{ip_range}.0/16"
            vuestorefront.vm.network "forwarded_port", guest: 22, host: Random.new.rand(1000...5000), id: 'ssh', auto_correct: true
            vuestorefront.vm.hostname = "#{name}"
            vuestorefront.vm.communicator = 'docker'
            vuestorefront.vm.provider 'docker' do |d|
                d.build_dir = "#{vagrant_root}/sites/vue-storefront/"
                d.dockerfile = "docker/vue-storefront/Dockerfile"
                d.has_ssh = false
                d.name = "frontend-#{name}"
                d.remains_running = true
                d.volumes = [
                    "#{vagrant_root}/sites/vue-storefront/babel.config.js:/var/www/babel.config.js",
                    "#{vagrant_root}/sites/vue-storefront/config:/var/www/config",
                    "#{vagrant_root}/sites/vue-storefront/core:/var/www/core",
                    "#{vagrant_root}/sites/vue-storefront/ecosystem.json:/var/www/ecosystem.json",
                    "#{vagrant_root}/sites/vue-storefront/.eslintignore:/var/www/.eslintignore",
                    "#{vagrant_root}/sites/vue-storefront/.eslintrc.js:/var/www/.eslintrc.js",
                    "#{vagrant_root}/sites/vue-storefront/lerna.json:/var/www/lerna.json",
                    "#{vagrant_root}/sites/vue-storefront/tsconfig.json:/var/www/tsconfig.json",
                    "#{vagrant_root}/sites/vue-storefront/tsconfig-build.json:/var/www/tsconfig-build.json",
                    "#{vagrant_root}/sites/vue-storefront/shims.d.ts:/var/www/shims.d.ts",
                    "#{vagrant_root}/sites/vue-storefront/package.json:/var/www/package.json",
                    "#{vagrant_root}/sites/vue-storefront/src:/var/www/src",
                    "/tmp/vuestorefront_#{name}:/var/www/dist",
                    "#{vagrant_root}/sites/vue-storefront/config/local.json.#{name}.#{dev_suffix}:/var/www/config/local.json"
                    ]
                d.env = { "BIND_HOST" => "0.0.0.0",
                          "NODE_CONFIG_ENV" => "docker",
                          "VS_ENV" => "#{mode}",
                          "PM2_ARGS" => "--no-daemon",
                          "NODE_TLS_REJECT_UNAUTHORIZED" => "0"
                        }
            end
        end
    end
    config.vm.define "broker-pwa", primary: false do |broker|
        broker.hostmanager.aliases =  [ "broker."+dev_domain, "broker.pwa."+dev_domain ]
        broker.vm.network "forwarded_port", guest: 22, host: Random.new.rand(1000...5000), id: 'ssh', auto_correct: true
        broker.vm.network :private_network, ip: "#{ip_range}.2", subnet: "#{ip_range}.0/16"
        broker.vm.hostname = "broker-pwa"
        broker.ssh.username = "vagrant"
        broker.ssh.password = "vagrant"
        broker.ssh.keys_only = false
        broker.trigger.after :up do |trigger|
            trigger.run = {inline: "bash -c 'vagrant hostmanager --provider docker'"}
        end
        broker.vm.provider 'docker' do |d|
            d.image = "enjo/ubuntu-devbox:latest"
            d.has_ssh = true
            d.name = "broker-pwa"
            d.remains_running = true
            d.volumes = [ENV['HOME']+"/.ssh/:/home/vagrant/.ssh"]
        end
    end

    config.vm.define "database-pwa", primary: false do |database|
        database.hostmanager.aliases = [ "database."+dev_domain, "database.pwa."+dev_domain ]
        database.vm.network :private_network, ip: "#{ip_range}.3", subnet: "#{ip_range}.0/16"
        database.vm.hostname = "database-pwa"
        database.vm.communicator = 'docker'
        database.vm.provider 'docker' do |d|
            d.image = "mariadb:10.2"
            d.has_ssh = false
            d.name = "database-pwa"
            d.remains_running = true
            d.volumes = ["#{persistent_storage}/mysql:/var/lib/mysql"]
            d.env = { "MYSQL_ROOT_PASSWORD" => "#{mysql_password}" }
        end
    end

    config.vm.define "redis-pwa", primary: false do |redis|
        redis.hostmanager.aliases = [ "redis."+dev_domain, "redis.pwa."+dev_domain ]
        redis.vm.network :private_network, ip: "#{ip_range}.4", subnet: "#{ip_range}.0/16"
        redis.vm.network "forwarded_port", guest: 6379, host: 6379, auto_correct: true
        redis.vm.hostname = "redis-pwa"
        redis.vm.communicator = 'docker'
        redis.vm.provider 'docker' do |d|
            d.build_dir = "#{vagrant_root}/Docker/redis"
            d.has_ssh = false
            d.name = "redis-pwa"
            d.remains_running = true
        end
    end

    config.vm.define "elasticsearch-pwa", primary: false do |elasticsearch|
        elasticsearch.hostmanager.aliases = [ "elasticvuesf."+dev_domain, "elasticvuesf.pwa."+dev_domain, "elasticsearch.pwa."+dev_domain  ]
        elasticsearch.vm.network :private_network, ip: "#{ip_range}.5", subnet: "#{ip_range}.0/16"
        elasticsearch.vm.hostname = "elasticsearch-pwa"
        elasticsearch.vm.communicator = 'docker'
        #elasticsearch.vm.provision "file", source: "#{vagrant_root}/elasticsearch.yml", destination: "/etc/elasticsearch/elasticsearch.yml"

        elasticsearch.vm.provider 'docker' do |d|
            d.image = "docker.elastic.co/elasticsearch/elasticsearch:7.8.0"
            d.has_ssh = true
            d.name = "elasticsearch-pwa"
            d.remains_running = true
            d.volumes = [
                "#{persistent_storage}/elasticsearch:/usr/share/elasticsearch/data",
                "#{vagrant_root}/elasticsearch.yml:/etc/elasticsearch/elasticsearch.yml"
            ]
            d.env = { "discovery.type" => "single-node" }
        end
    end


    config.vm.define "kibana-pwa", primary: false do |kibana|
        kibana.hostmanager.aliases =  [ "kibana."+dev_domain, "kibana.pwa."+dev_domain ]
        kibana.vm.network "forwarded_port", guest: 22, host: Random.new.rand(1000...5000), id: 'ssh', auto_correct: true
        kibana.vm.network :private_network, ip: "#{ip_range}.6", subnet: "#{ip_range}.0/16"
        kibana.vm.hostname = "kibana-pwa"
        kibana.vm.communicator = 'docker'
        kibana.vm.provider 'docker' do |d|
            d.image = "docker.elastic.co/kibana/kibana:7.8.0"
            d.has_ssh = true
            d.name = "kibana-pwa"
            d.remains_running = true
            d.env = {
                "ELASTICSEARCH_URL" => "http://elasticsearch.pwa."+dev_domain+":9200",
                "ELASTICSEARCH_HOSTS" => "http://elasticsearch.pwa."+dev_domain+":9200"
            }
        end
    end

    config.vm.define "magento-pwa", primary: true do |magento|
        magento.hostmanager.aliases = [ "magento."+dev_domain, "magento.pwa."+dev_domain, "magento.sante.pwa."+dev_domain ]
        magento.vm.provision "file", source: "#{vagrant_root}/magento.nginx.conf", destination: "/tmp/magento"
        magento.vm.provision "shell" do |s|
            s.path = "bootstrap.sh"
            s.args = "#{dev_domain} #{ip_range}.200"
        end
        if File.exist?("provision/bootstrap.sh")
            magento.vm.provision "shell" do |s|
                s.path = "provision/bootstrap.sh"
                s.args = "#{dev_domain} #{ip_range}.200"
            end
        end

        magento.ssh.username = "vagrant"
        magento.ssh.password = "vagrant"
        magento.ssh.keys_only = false
        magento.vm.network :private_network, ip: "#{ip_range}.200", subnet: "#{ip_range}.0/16"
        magento.vm.network "forwarded_port", guest: 22, host: 2230, id: 'ssh', auto_correct: true
        magento.vm.hostname = "magento-pwa"
        magento.vm.provider 'docker' do |d|
            #d.image = "proxiblue/magento2:latest"
            d.build_dir = "#{vagrant_root}/Docker/magento"
            d.has_ssh = true
            d.name = "magento-pwa"
            d.create_args = ["--cap-add=NET_ADMIN"]
            d.remains_running = true
            d.volumes = ["/tmp/.X11-unix:/tmp/.X11-unix", ENV['HOME']+"/.ssh/:/home/vagrant/.ssh", "#{persistent_storage}/composer:/home/vagrant/.composer"]
            d.env = { "DEV_DOMAIN" => "#{dev_domain}", "WEB_IP" => "#{ip_range}.200" }
        end
    end
    if File.exist?("#{vagrant_root}/reverseproxy/nginx.conf")
        reverseproxynodes.each do |name, ip|
            config.vm.define "reverseproxy-#{name}", primary: false do |reverseproxy|
                reverseproxy.hostmanager.aliases = [ "#{name}."+dev_domain, "api.#{name}."+dev_domain  ]
                reverseproxy.vm.network :private_network, ip: "#{ip_range}.#{ip}", subnet: "#{ip_range}.0/16"
                reverseproxy.vm.network "forwarded_port", guest: 22, host: Random.new.rand(1000...5000), id: 'ssh', auto_correct: true
                reverseproxy.vm.hostname = "reverseproxy-#{name}"
                reverseproxy.vm.provision "shell" do |s|
                    s.path = "#{vagrant_root}/reverseproxy/bootstrap.sh"
                    s.args = "#{name} #{dev_domain}"
                end
                reverseproxy.ssh.username = "vagrant"
                reverseproxy.ssh.password = "vagrant"
                reverseproxy.ssh.keys_only = false
                reverseproxy.vm.provider 'docker' do |d|
                    d.build_dir = "#{vagrant_root}/Docker/nginx"
                    #d.image = "nginx:latest"
                    d.has_ssh = true
                    d.name = "reverseproxy-#{name}"
                    d.remains_running = true
                    d.volumes = [
                        "#{vagrant_root}/reverseproxy/nginx.conf:/tmp/nginx.conf:ro",
                        "#{vagrant_root}/Docker/magento/common/nginx/ssl:/etc/nginx/ssl"
                    ]
                end
                ## FINAL BOX MUST HAVE THIS
                reverseproxy.trigger.after :up do |trigger|
                        trigger.run = {inline: "bash -c 'vagrant hostmanager --provider docker'"}
                end
                ##
                ## FINAL BOX MUST HAVE THIS
                reverseproxy.trigger.before :up do |trigger|
                        trigger.run = {inline: "bash -c 'vagrant hostmanager --provider docker'"}
                end
                ##
            end
        end
    end
end
