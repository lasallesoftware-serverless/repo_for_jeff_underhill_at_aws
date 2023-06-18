AWS REGIONS: EC2 + Lambda Layers/Functions in the **same** AWS region
---

- set-up and run PHP from Lambda in console
- use AWS sample bootstrap file
- use AL2 on EC2
- assume that IAM is set up
- create EC2 instance
- ssh into EC2 instance

```bash
sudo yum update -y


sudo rm -rf /usr/lib/python2.7/site-packages/awscli/
sudo rm -rf /usr/lib/python2.7/site-packages/botocore/
sudo rm /usr/bin/aws
sudo rm /usr/bin/aws_completer
sudo rm /usr/share/zsh/site-functions/aws_zsh_completer.sh
sudo rm /etc/bash_completion.d/aws_bash_completer
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip -q -o awscliv2.zip
sudo ./aws/install --update

# clean-up
# -f to suppress the confirmation prompt
sudo rm -rf aws
sudo rm awscliv2.zip -f

# install stuff needed for compiling PHP
sudo yum install autoconf bison gcc gcc-c++ re2c libcurl-devel libxml2-devel openssl-devel sqlite-devel git -y

# grab the openssl version that works with AL2
curl -sL http://www.openssl.org/source/openssl-1.0.1k.tar.gz | tar -xvz
cd openssl-1.0.1k
./config && make && sudo make install
cd ~

# grab PHP's source code
git clone https://github.com/php/php-src.git
cd php-src
git checkout PHP-8.2.7

# start compiling PHP
./buildconf --force

./configure --prefix=/home/ec2-user/runtime/ \
   --disable-cgi \
   --disable-phpdbg \
   --with-openssl=/usr/local/ssl \
   --with-curl \
   --with-zlib  
   
make -j$(($(nproc) + 1))
 
make install
   
# skip the PHP tests
# sapi/cli/php run-tests.php -j$(($(nproc) + 1))
   
# grab the AWS sample bootstrap for PHP   
wget https://raw.githubusercontent.com/aws-samples/php-examples-for-aws-lambda/master/0.1-SimplePhpFunction/bootstrap
# make executable
chmod +x bootstrap

# create the runtime file that will be uploaded to Lambda
zip -r runtime.zip bin bootstrap

# upload the runtime zip file to Lambda
# your region may differ
/usr/local/bin/aws lambda publish-layer-version \
    --description "PHP runtime v8.2.7" \
    --license-info "MIT" \
    --layer-name PHP-827 \
    --zip-file fileb://runtime.zip \
    --region ca-central-1


# create a separate Lambda Layer for the composer installable stuff specifically for our compiled PHP runtime
./bin/php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
./bin/php -r "if (hash_file('sha384', 'composer-setup.php') === '55ce33d7678c5a611085589f1f3ddf8b3c52d662cd01d4ba75c0ee0459970c2200a51f492d557530c71c15d8dba01eae') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
php composer-setup.php
./bin/php composer-setup.php
./bin/php -r "unlink('composer-setup.php');"

# guzzle is for API calls
./bin/php composer.phar require guzzlehttp/guzzle

# create the zip file
zip -r vendor.zip vendor/

# upload to Lambda
# your region may differ
/usr/local/bin/aws lambda publish-layer-version \
    --description "Composer's vendor folder for PHP runtime" \
    --license-info "MIT" \
    --layer-name composer-vendor-folder-for-php-runtime \
    --zip-file fileb://vendor.zip \
    --region ca-central-1
     
```


