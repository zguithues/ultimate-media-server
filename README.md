# Ultimate Torrent Setup

This is a repository of all the files you need to configure the ultimate torrent setup.  
Refer to the wiki for instructions: <a href="https://github.com/xombiemp/ultimate-torrent-setup/wiki" target="_blank">Ultimate Torrent Setup Wiki</a>

<a id="user-content-ultimate-torrent-setup" class="anchor" href="#ultimate-torrent-setup" aria-hidden="true"><span class="octicon octicon-link"></span></a>Ultimate Torrent Setup</h1>

<h2>
<a id="user-content-introduction" class="anchor" href="#introduction" aria-hidden="true"><span class="octicon octicon-link"></span></a>Introduction</h2>

<h3>
<a id="user-content-assumptions" class="anchor" href="#assumptions" aria-hidden="true"><span class="octicon octicon-link"></span></a>Assumptions</h3>

<p>This guide will help you setup the ultimate automatic torrent downloading solution. You will setup and configure, on Ubuntu server 14.04, rTorrent+ruTorrent, Sonarr, CouchPotato and Apache as a reverse proxy for all these. This guide assumes that you would like to be able to access all these interfaces from anywhere on the Internet (using a login form secured with an SSL certificate) and that you have a domain name you will use. This guide includes making a self-signed certificate if you don't have or don't want to purchase an SSL cert. Just keep in mind that you will get a certificate warning until you add a permanent exception in your browser. I would suggest using a certificate that is signed by a legitimate Certificate Authority. I personally use one that I got absolutely free from <a href="http://www.startssl.com" target="_blank">StartSSL</a>.</p>

<h3>
<a id="user-content-how-it-works" class="anchor" href="#how-it-works" aria-hidden="true"><span class="octicon octicon-link"></span></a>How it Works</h3>

<p>We will be using the ruTorrent plugins autotools, ratio &amp; extratio to accomplish what we want. The way these plugins work is best described by an example:<br>
Say you have a directory /torrent for your torrent downloads. What you would do is create three sub dirs in that directory called watch, download and complete. Inside these directories will contain a mirrored directory structure for the different categories of torrents you download. So your structure might look like this:<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/directories.png"><br>
So if you drop a torrent into /torrent/watch/tv/sonarr, the torrent download is started in /torrent/download/tv/sonarr. Also a label is automatically applied to the torrent based on the directory: in this example our torrent would receive the label tv/sonarr. Once the torrent is finished downloading, a hard link will be created in /torrent/complete/tv/sonarr. A hard link means that a duplicate file is made, but it shares the same data as the original. This is better than copying because it is instant and you don't have to duplicate the data on your hard drive. So now if we just have Sonarr watch the /torrent/complete/tv/sonarr directory it can process the hard linked files and move them to a final TV directory, AND we can continue to seed with the original file in /torrent/download/tv/sonarr with no problems. Even when we delete the file in the download directory, it won't affect the other hard link in the place where Sonarr placed it.<br>
We'll also create some Ratio Rules that will seed a torrent of a specific label to a specified ratio and then automatically delete the torrent. So you won't have to monitor your torrents and manually clean them up when they've seeded enough, they will automatically seed to the perfect amount and then delete themselves.<br>
The end result will be: you add a show to Sonarr, Sonarr will automatically search for new episodes and download the torrent file as soon as it's available. The torrent will be added and started in rTorrent. As soon as the download is complete, Sonarr will process the hard link of the data by renaming it nicely and placing it in a nice, organized TV Shows folder. Meanwhile rTorrent will still be seeding the original file until it reaches the defined ratio, and then will auto delete the torrent and the original file from the download dir.  100% automated TV downloading! Couchpotato does the same thing for movies.</p>

<h2>
<a id="user-content-install-rtorrent-rutorrent-sonarr-couchpotato" class="anchor" href="#install-rtorrent-rutorrent-sonarr-couchpotato" aria-hidden="true"><span class="octicon octicon-link"></span></a>Install rTorrent, ruTorrent, Sonarr, CouchPotato</h2>

<p>Login as your user then execute these commands one line at a time.</p>

<p>Update Ubuntu</p>

<pre><code>sudo apt-get update

sudo apt-get dist-upgrade

</code></pre>

<p>Install Prereqs</p>

<pre><code>sudo apt-get install -y git-core subversion build-essential automake libtool libcppunit-dev libcurl4-openssl-dev libsigc++-2.0-dev libncurses5-dev zip rar unrar apache2 apache2-utils php5 php5-curl php5-geoip mediainfo libav-tools

</code></pre>

<p>Download the configs repo, which contain the necessary config files you will need.</p>

<pre><code>cd ~

git clone https://github.com/xombiemp/ultimate-torrent-setup.git configs

</code></pre>

<h3>
<a id="user-content-rtorrent" class="anchor" href="#rtorrent" aria-hidden="true"><span class="octicon octicon-link"></span></a>rTorrent</h3>

<h4>
<a id="user-content-install-xmlrpc-from-svn" class="anchor" href="#install-xmlrpc-from-svn" aria-hidden="true"><span class="octicon octicon-link"></span></a>Install xmlrpc from svn</h4>

<pre><code>cd /usr/local/src

sudo svn checkout http://svn.code.sf.net/p/xmlrpc-c/code/advanced xmlrpc-c

cd xmlrpc-c

sudo ./configure

sudo make

sudo make install

</code></pre>

<h4>
<a id="user-content-install-libtorrent" class="anchor" href="#install-libtorrent" aria-hidden="true"><span class="octicon octicon-link"></span></a>Install libtorrent</h4>

<pre><code>cd /usr/local/src

sudo git clone https://github.com/rakshasa/libtorrent.git

cd libtorrent

sudo ./autogen.sh

sudo ./configure

sudo make

sudo make install

</code></pre>

<h4>
<a id="user-content-install-rtorrent" class="anchor" href="#install-rtorrent" aria-hidden="true"><span class="octicon octicon-link"></span></a>Install rTorrent</h4>

<pre><code>cd /usr/local/src

sudo git clone https://github.com/rakshasa/rtorrent.git

cd rtorrent

sudo ./autogen.sh

sudo ./configure --with-xmlrpc-c

sudo make

sudo make install

sudo ldconfig

</code></pre>

<h4>
<a id="user-content-make-directories" class="anchor" href="#make-directories" aria-hidden="true"><span class="octicon octicon-link"></span></a>Make Directories</h4>

<p>Create the config directory for rTorrent</p>

<pre><code>mkdir -p ~/.config/rtorrent

</code></pre>

<p>Create and change directory to where you are going to download all your torrents to. For this guide we are using /data.</p>

<pre><code>sudo mkdir /data

cd /data

</code></pre>

<p>Then execute this command which will create the folder structure I illustrated earlier:</p>

<pre><code>sudo mkdir -p torrent/{complete/{movie/couchpotato,music,tv/sonarr,game,book,software,other},download/{movie/couchpotato,music,tv/sonarr,game,book,software,other},watch/{movie/couchpotato,music,tv/sonarr,game,book,software,other}} Media/{Movies,'TV Shows'}

</code></pre>

<p>Give these folders full permissions by executing this on the parent directory:</p>

<pre><code>sudo chmod -R 777 /data

</code></pre>

<p>or whatever your path is.</p>

<h4>
<a id="user-content-make-config-files-and-boot-script" class="anchor" href="#make-config-files-and-boot-script" aria-hidden="true"><span class="octicon octicon-link"></span></a>Make Config Files and Boot Script</h4>

<p>Make rTorrent config file:</p>

<pre><code>cd ~

mv ~/configs/rtorrent.rc ~/.config/rtorrent/

</code></pre>

<p>Edit <em>~/.config/rtorrent/rtorrent.rc</em> and look for where it says </p>

<blockquote>
<p>/CHANGE/THIS/PATH </p>
</blockquote>

<p>for the directory path and edit the path for your setup. In our example, the directory path is the download directory we created previously, /data/torrent/download.
Look for </p>

<blockquote>
<p>YOURUSER </p>
</blockquote>

<p>in the session path and change it to your username.<br>
Also look for </p>

<blockquote>
<p>APACHE_WEBAUTH_USER </p>
</blockquote>

<p>at the bottom and change it to the user that you want to use for Apache web authentication (we haven't created the login yet but we will later). Change any other random options that you want.</p>

<pre><code>sudo mv ~/configs/rtorrent.conf /etc/init/

</code></pre>

<p>Edit <em>/etc/init/rtorrent.conf</em> and change the three occurrences of the following to your user:</p>

<blockquote>
<p>YOURUSER</p>
</blockquote>

<h3>
<a id="user-content-rutorrent" class="anchor" href="#rutorrent" aria-hidden="true"><span class="octicon octicon-link"></span></a>ruTorrent</h3>

<h4>
<a id="user-content-install-rutorrent" class="anchor" href="#install-rutorrent" aria-hidden="true"><span class="octicon octicon-link"></span></a>Install ruTorrent</h4>

<p>The script update-rutorrent will install ruTorrent and any plugins you want automatically. Then whenever you run it again it will update ruTorrent and any plugins that have updates.</p>

<pre><code>sudo mv ~/configs/update-rutorrent /usr/local/bin/

sudo chmod +x /usr/local/bin/update-rutorrent

</code></pre>

<p>Edit <em>/usr/local/bin/update-rutorrent</em> and change the webroot_path if it is different for you (probably not).
Edit the plugins variable to include or exclude any plugins that you want. You can leave it as it is to use the plugins that I use. The ones you have to keep are autotools, ratio, extratio and throttle.
Once all the edits are made to the script, run it to install ruTorrent and your plugins:</p>

<pre><code>sudo update-rutorrent

</code></pre>

<h4>
<a id="user-content-edit-rutorrent-configs" class="anchor" href="#edit-rutorrent-configs" aria-hidden="true"><span class="octicon octicon-link"></span></a>Edit ruTorrent configs</h4>

<p>Edit <em>/var/www/rutorrent/conf/config.php</em>
Change $topDirectory to the top directory where your torrents are downloaded. In our example:</p>

<blockquote>
<p>$topDirectory = '/data/';</p>
</blockquote>

<p>Change:</p>

<blockquote>
<p>$saveUploadedTorrents = false;</p>
</blockquote>

<p>Edit <em>/var/www/rutorrent/plugins/autotools/conf.php</em>
The $autowatch_interval variable is how often in seconds it looks in the watch folders for new torrent files. I changed mine to:</p>

<blockquote>
<p>$autowatch_interval = 60;</p>
</blockquote>

<p>Edit <em>/var/www/rutorrent/plugins/fileshare/conf.php</em>
Set the $downloadpath variable to match the domain you will use to access your site:</p>

<blockquote>
<p>$downloadpath = '<a href="http://yourdomain.com/public/share.php">http://yourdomain.com/public/share.php</a>';</p>
</blockquote>

<p>Edit <em>/var/www/rutorrent/plugins/screenshots/conf.php</em>
Set the following variable value:</p>

<blockquote>
<p>$pathToExternals['ffmpeg'] = '/usr/bin/avconv';</p>
</blockquote>

<p>Edit <em>/var/www/rutorrent/plugins/unpack/conf.php</em>
Set the following variable values:</p>

<blockquote>
<p>$deleteAutoArchives = true;<br>
$unpackToTemp = true;</p>
</blockquote>

<h4>
<a id="user-content-configure-apache" class="anchor" href="#configure-apache" aria-hidden="true"><span class="octicon octicon-link"></span></a>Configure Apache</h4>

<h6>
<a id="user-content-make-apache-config-file" class="anchor" href="#make-apache-config-file" aria-hidden="true"><span class="octicon octicon-link"></span></a>Make Apache config file</h6>

<pre><code>sudo mv ~/configs/apache-portal.conf /etc/apache2/sites-available/

sudo a2dissite 000-default.conf

sudo a2ensite apache-portal.conf

</code></pre>

<p>Edit <em>/etc/apache2/sites-available/apache-portal.conf</em> and find all four instances of</p>

<blockquote>
<p>SessionCryptoPassphrase CHANGEME</p>
</blockquote>

<p>and change CHANGEME to the same unique passphrase of your choosing. You won't need to use it, it's just for encrypting the session cookie.</p>

<h6>
<a id="user-content-enable-necessary-modules" class="anchor" href="#enable-necessary-modules" aria-hidden="true"><span class="octicon octicon-link"></span></a>Enable necessary modules</h6>

<pre><code>sudo a2enmod rewrite request headers proxy_http auth_form session_cookie session_crypto ssl

</code></pre>

<h6>
<a id="user-content-setup-default-site-files" class="anchor" href="#setup-default-site-files" aria-hidden="true"><span class="octicon octicon-link"></span></a>Setup default site files</h6>

<pre><code>sudo mv ~/configs/site/* /var/www/

sudo mkdir /var/www/public

sudo ln -s /var/www/rutorrent/plugins/fileshare/share.php /var/www/public/

sudo chown -R www-data:www-data /var/www/*

</code></pre>

<h6>
<a id="user-content-create-password-for-apache-authentication" class="anchor" href="#create-password-for-apache-authentication" aria-hidden="true"><span class="octicon octicon-link"></span></a>Create password for Apache authentication</h6>

<pre><code>sudo htpasswd -c /etc/apache2/passwd APACHE_WEBAUTH_USER

</code></pre>

<p>Replace APACHE_WEBAUTH_USER in the command with the same username as you put in the bottom of rtorrent.rc.</p>

<h6>
<a id="user-content-create-self-signed-ssl-cert-you-can-skip-this-step-if-you-are-using-a-signed-cert" class="anchor" href="#create-self-signed-ssl-cert-you-can-skip-this-step-if-you-are-using-a-signed-cert" aria-hidden="true"><span class="octicon octicon-link"></span></a>Create self-signed SSL cert (you can skip this step if you are using a signed cert)</h6>

<p>Edit <em>/etc/ssl/openssl.cnf</em> and uncomment the line that says:</p>

<blockquote>
<p>req_extensions = v3_req # The extensions to add to a certificate request</p>
</blockquote>

<p>Go the v3_req section and add subjectAltName variable at the bottom so it looks like</p>

<blockquote>
<p>[ v3_req ]</p>

<p># Extensions to add to a certificate request</p>

<p>basicConstraints = CA:FALSE<br>
keyUsage = nonRepudiation, digitalSignature, keyEncipherment<br>
subjectAltName = @alt_names</p>
</blockquote>

<p>Under that section add another section that lists your domain name so it looks like</p>

<blockquote>
<p>[ alt_names ]<br>
DNS.1 = yourdomain.com<br>
DNS.2 = *.yourdomain.com</p>
</blockquote>

<p>Now generate your certificate:</p>

<pre><code>sudo openssl req -new -newkey rsa:2048 -x509 -extensions v3_req -days 36500 -nodes -out /etc/apache2/ssl.pem -keyout /etc/apache2/ssl.pem

</code></pre>

<p>You’ll now have to fill out some info about your certificate.  This is an example:</p>

<blockquote>
<p>Country Name (2 letter code) [AU]:US<br>
State or Province Name (full name) [Some-State]:California<br>
Locality Name (eg, city) []:Los Angeles<br>
Organization Name (eg, company) [Internet Widgits Pty Ltd]:FakeName<br>
Organizational Unit Name (eg, section) []:IT<br>
Common Name (e.g. server FQDN or YOUR name) []:FakeName<br>
Email Address []:  </p>
</blockquote>

<h6>
<a id="user-content-configure-some-apache-and-php-default-values" class="anchor" href="#configure-some-apache-and-php-default-values" aria-hidden="true"><span class="octicon octicon-link"></span></a>Configure some Apache and PHP default values</h6>

<p>Edit <em>/etc/apache2/apache2.conf</em> and add the directive for your server’s hostname:</p>

<blockquote>
<p>ServerName “hostname”</p>
</blockquote>

<p>Edit <em>/etc/php5/apache2/php.ini</em> and search for the line that contains date.timezone =.  Set the value for your timezone from <a href="http://www.php.net/timezones" target="_blank">PHP Timezones</a>. Mine looks like:</p>

<blockquote>
<p>date.timezone = America/Denver</p>
</blockquote>

<p>Edit <em>/etc/php5/cli/php.ini</em> and do the same thing as above.</p>

<h3>
<a id="user-content-sonarr" class="anchor" href="#sonarr" aria-hidden="true"><span class="octicon octicon-link"></span></a>Sonarr</h3>

<h4>
<a id="user-content-install-sonarr-bootscripts-and-configure" class="anchor" href="#install-sonarr-bootscripts-and-configure" aria-hidden="true"><span class="octicon octicon-link"></span></a>Install Sonarr, Bootscripts and Configure</h4>

<pre><code>cd ~

sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys FDA5DFFC

sudo echo "deb http://apt.sonarr.tv/ master main" | sudo tee /etc/apt/sources.list.d/sonarr.list

sudo apt-get update

sudo apt-get install -y nzbdrone

sudo mv ~/configs/sonarr.conf /etc/init/

</code></pre>

<p>Edit <em>/etc/init/sonarr.conf</em> and change the following to your user name:</p>

<blockquote>
<p>setuid YOURUSER</p>
</blockquote>

<h3>
<a id="user-content-couchpotato" class="anchor" href="#couchpotato" aria-hidden="true"><span class="octicon octicon-link"></span></a>CouchPotato</h3>

<h4>
<a id="user-content-install-couchpotato-bootscripts-and-configure" class="anchor" href="#install-couchpotato-bootscripts-and-configure" aria-hidden="true"><span class="octicon octicon-link"></span></a>Install CouchPotato, Bootscripts and Configure</h4>

<pre><code>cd ~

git clone https://github.com/RuudBurger/CouchPotatoServer.git .couchpotato

sudo cp ~/.couchpotato/init/ubuntu /etc/init.d/couchpotato

sudo chmod +x /etc/init.d/couchpotato

sudo update-rc.d couchpotato defaults

mkdir ~/.config/couchpotato

sudo mv ~/configs/couchpotato /etc/default/

sudo rm -r ~/configs/

</code></pre>

<p>Edit <em>/etc/default/couchpotato</em> and change the following to your user name:</p>

<blockquote>
<p>CP_USER=YOURUSER </p>
</blockquote>

<h3>
<a id="user-content-finish-up" class="anchor" href="#finish-up" aria-hidden="true"><span class="octicon octicon-link"></span></a>Finish up</h3>

<pre><code>sudo service apache2 restart

sudo service rtorrent restart

sudo service sonarr start

sudo service couchpotato start

sudo service sonarr stop

sudo service couchpotato stop

</code></pre>

<p>Edit <em>~/.config/NzbDrone/config.xml</em> and change:</p>

<blockquote>
<p>&lt;UrlBase&gt;/sonarr&lt;/UrlBase&gt;</p>
</blockquote>

<p>Edit <em>~/.config/couchpotato/settings.conf</em> and change:</p>

<blockquote>
<p>url_base = /couchpotato</p>
</blockquote>

<pre><code>sudo service sonarr start

sudo service couchpotato start

</code></pre>

<h2>
<a id="user-content-configure-the-gui-settings" class="anchor" href="#configure-the-gui-settings" aria-hidden="true"><span class="octicon octicon-link"></span></a>Configure the GUI Settings</h2>

<h3>
<a id="user-content-rutorrent-1" class="anchor" href="#rutorrent-1" aria-hidden="true"><span class="octicon octicon-link"></span></a>ruTorrent</h3>

<p>First browse to <a href="https://yourdomain.com/rutorrent">https://yourdomain.com/rutorrent</a><br>
You will be presented with the login window:<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/rutorrent/Login.png"><br>
Login with the username and password you created earlier.</p>

<p>Click on the gear cog to enter the settings.<br>
In Autotools enable all the options and browse to your complete and watch directories. Also, be sure to select Hard Link for the operation type:<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/rutorrent/Autotools.png"></p>

<p>In History I only check Deletion. This is nice because Sonarr could download a torrent, seed it to 2.0, and delete it all while you are sleeping. So the History plugin will let you see a record of when it was deleted and what your ratio was, just for your sanity:<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/rutorrent/History.png"></p>

<p>In Unpack check enable autounpacking:<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/rutorrent/Unpack.png"></p>

<p>In Ratio Groups set up a default group and a Sonarr group:<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/rutorrent/RatioGroups.png"></p>

<p>Click ok. Then click on the wrench/screwdriver next to the gear cog and click on Ratio Rules. Set up a rule for Sonarr:<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/rutorrent/RatioRules.png"><br>
You can setup another Ratio Rule for Couchpotato downloads too, if you want. Alternatively, you could set up ratio rules for each individual tracker by selecting "One of torrent's tracker URLs contain" from the drop down list.</p>

<p>Now when you add a torrent you just click on the directory button and choose which category it is. Then it will automatically receive that label and it will download to the proper directory.<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/rutorrent/AddTorrent.png"></p>

<h3>
<a id="user-content-sonarr-1" class="anchor" href="#sonarr-1" aria-hidden="true"><span class="octicon octicon-link"></span></a>Sonarr</h3>

<p>Now browse to <a href="https://yourdomain.com/sonarr">https://yourdomain.com/sonarr</a><br>
Click on Settings and you'll see the Media Managment tab.<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/sonarr/MediaManagement.png"><br>
Move the switch to Shown next to Advanced Settings. Switch Rename Episodes to Yes. Set the renaming formats to the way you want. Mine are set to conform to Plex naming standards. Click the Save button.</p>

<p>Click on the Indexers tab.<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/sonarr/Indexers.png"><br>
Click the plus button to add your desired providers. Configure the provider with the info it asks for, which you will usually find in your respective rss feed url for the tracker. If your tracker does not support search, I would suggest checking out <a href="https://github.com/zone117x/Jackett" target="_blank">Jackett</a>. It is a separate webapp that adds search capabilities to many trackers. I haven't used it personally, but I've heard it works well.</p>

<p>Click on the Download Client tab.<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/sonarr/DownloadClient.png"><br>
Click the plus sign and click Torrent Blackhole. For the Torrent Folder browse to your sonarr watch directory. In our example it would be /data/torrent/watch/tv/sonarr. For the Watch Folder browse to your sonarr complete folder. In our example it would be /data/torrent/complete/tv/sonarr.</p>

<p>Click on Series -&gt; Add Series -&gt; Import Existing Series On Disk<br>
Browse to your TV Shows folder. In our example it would be /data/Media/TV Shows/<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/sonarr/SelectFolder.png"><br>
Click the close button.</p>

<h3>
<a id="user-content-couchpotato-1" class="anchor" href="#couchpotato-1" aria-hidden="true"><span class="octicon octicon-link"></span></a>CouchPotato</h3>

<p>Now browse to <a href="https://yourdomain.com/couchpotato">https://yourdomain.com/couchpotato</a><br>
You'll see a Wizard when you first login. Just click the Finish button; we'll configure the settings manually.</p>

<p>Click the gear cog and click settings.</p>

<p><img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/couchpotato/General-cp.png"><br>
Uncheck Launch the browser when I start.</p>

<p>Click on Searcher, check the box for Advanced Settings<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/couchpotato/Searcher.png"><br>
Change the search frequency if you want.</p>

<p><img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/couchpotato/Providers-cp.png"><br>
Configure your providers.</p>

<p>Click Categories<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/couchpotato/Categories.png"><br>
You can create a category like the above that requires the source to be Blu-Ray. </p>

<pre><code>bluray, blu-ray, blu &amp; ray
</code></pre>

<p>Click on Qualities<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/couchpotato/Sizes.png"><br>
I've found that these sizes work well. I've missed some downloads when I used the default values.</p>

<p>Click Downloaders<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/couchpotato/Downloaders.png"><br>
Click Black hole and browse to your watch couchpotato directory.</p>

<p>Click Renamer and match my settings<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/couchpotato/Renamer.png"></p>

<p>Click Metadata and match my settings<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/couchpotato/Metadata-cp.png"></p>

<h2>
<a id="user-content-conclusion" class="anchor" href="#conclusion" aria-hidden="true"><span class="octicon octicon-link"></span></a>Conclusion</h2>

<p>Now that everything is configured you can add TV shows to Sonarr and movies to Couchpotato and watch the magic. You can update Sonarr using apt-get. CouchPotato will notify you in it's interface when it has updates available.  You can keep ruTorrent up to date by running the update-rutorrent script again. If rTorrent releases an update that you want to install, just follow the install sections of this guide again with the new files for xmlrpc, libtorrent and rTorrent.</p>

<p>Enjoy 100% automated TV and movie downloading!</p>

<h2>
<a id="user-content-optional-extras" class="anchor" href="#optional-extras" aria-hidden="true"><span class="octicon octicon-link"></span></a>Optional Extras</h2>

<p>These are some additional things you can do that make this setup even better.</p>

<h3>
<a id="user-content-notifications" class="anchor" href="#notifications" aria-hidden="true"><span class="octicon octicon-link"></span></a>Notifications</h3>

<p>Both Sonarr and CouchPotato have the ability to notify you when they start a download and finish a download. My favorite provider, and the one I use, is <a href="https://pushover.net" target="_blank">Pushover</a>. Pushover supports both iOS and Android and is simply a one time purchase rather than a monthly subscription.</p>

<h4>
<a id="user-content-sonarr-2" class="anchor" href="#sonarr-2" aria-hidden="true"><span class="octicon octicon-link"></span></a>Sonarr</h4>

<p>Click Settings -&gt; Connect. Check Pushover (or your provider of choice) and configure the settings.<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/sonarr/Notifications.png"></p>

<h4>
<a id="user-content-couchpotato-2" class="anchor" href="#couchpotato-2" aria-hidden="true"><span class="octicon octicon-link"></span></a>CouchPotato</h4>

<p>Click the gear cog -&gt; Settings -&gt; Notifications. Check Pushover (or your provider of choice) and configure the settings.<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/couchpotato/Notifications.png"></p>

<p>Now you will receive a notification when ever a show or movie starts and finishes downloading.<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/pushover.png"></p>

<h3>
<a id="user-content-mobile-access" class="anchor" href="#mobile-access" aria-hidden="true"><span class="octicon octicon-link"></span></a>Mobile Access</h3>

<p>Now that everything is setup, you'll probably find yourself wanting to access these interfaces from your mobile device. I have an iPhone, so all the screenshots will be from that, but since these are all webapps, this should work from any device.</p>

<h4>
<a id="user-content-rutorrent-2" class="anchor" href="#rutorrent-2" aria-hidden="true"><span class="octicon octicon-link"></span></a>ruTorrent</h4>

<p>A mobile plugin was created by <a href="https://github.com/zebraxxl" target="_blank">zebraxxl</a>, but it appears he has abandoned it. I have created a fork of <a href="https://github.com/xombiemp/rutorrentMobile" target="_blank">rutorrentMobile</a> and made a number of enhancements.
The update-rutorrent script is set to automatically install this plugin so it should already be installed. Now from your mobile browser, browse to <a href="https://yourdomain.com/rutorrent">https://yourdomain.com/rutorrent</a> and you should see the mobile interface for rutorrent. Add to your mobile home screen to create a web app.<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/rutorrent/rutorrentMobile.png"></p>

<h4>
<a id="user-content-sonarr-3" class="anchor" href="#sonarr-3" aria-hidden="true"><span class="octicon octicon-link"></span></a>Sonarr</h4>

<p>Sonarr has a responsive design that is very functional on a mobile device. From your mobile browser, browse to <a href="https://yourdomain.com/sonarr">https://yourdomain.com/sonarr</a> and you should see the interface for Sonarr. Add to your mobile home screen to create a web app.<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/sonarr/SonarrMobile.jpg"></p>

<h4>
<a id="user-content-couchpotato-3" class="anchor" href="#couchpotato-3" aria-hidden="true"><span class="octicon octicon-link"></span></a>CouchPotato</h4>

<p>There is nothing special required for CouchPotato. Just from your mobile browser, browse to <a href="https://yourdomain.com/couchpotato">https://yourdomain.com/couchpotato</a> and you should see the mobile interface for CouchPotato. Add to your mobile home screen to create a web app.<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/couchpotato/couchpotatoMobile.png"></p>

<h3>
<a id="user-content-bittorrent-sync-watch-folders" class="anchor" href="#bittorrent-sync-watch-folders" aria-hidden="true"><span class="octicon octicon-link"></span></a>BitTorrent Sync watch folders</h3>

<p>If you haven't heard about <a href="http://www.bittorrent.com/sync" target="_blank">BitTorrent Sync</a> yet, go read about it. It is awesome! It's like Dropbox, but instead of syncing to the cloud first, the folders sync from Peer to Peer. We can set up BitTorrent Sync on our watch directory so that we are able to drop .torrent files into those directories from any device. On iOS for instance, you can download a .torrent file in Safari and then use the Open with... option to place the file in one of the watch folders, depending on the category. Then BitTorrent Sync will sync that file to all the Peers connected to it, includng our Ubuntu server, which means the .torrent file will be downloaded to the correct watch folder and picked up by ruTorrent immediately!</p>

<pre><code>cd ~

wget https://download-cdn.getsyncapp.com/stable/linux-x64/BitTorrent-Sync_x64.tar.gz

sudo tar xvzf BitTorrent-Sync_x64.tar.gz -C /usr/local/bin/ btsync

rm BitTorrent-Sync_x64.tar.gz

mkdir ~/.config/btsync

btsync --dump-sample-config &gt; ~/.config/btsync/config.json

sudo wget https://raw.github.com/xombiemp/ultimate-torrent-setup/master/btsync.conf -P /etc/init/

</code></pre>

<p>Edit <em>/etc/init/btsync.conf</em> and change the three occurrences of the following to your user name:</p>

<blockquote>
<p>YOURUSER</p>
</blockquote>

<p>Edit <em>~/.config/btsync/config.json</em>
Change the following to what you want:</p>

<blockquote>
<p>"device_name": "My Sync Device"</p>
</blockquote>

<p>Uncomment and change the following to /home/YOURUSER/.config/btsync/</p>

<blockquote>
<p>"storage_path" : "/home/user/.sync"</p>
</blockquote>

<p>Add the following to <em>/etc/apache2/sites-available/apache-portal.conf</em> under the &lt;VirtualHost *:443&gt; section and change the SessionCryptoPassphrase to what the others are.</p>

<pre><code>    &lt;Location /btsync&gt;
        Require valid-user
        AuthType form
        AuthFormProvider file
        AuthFormLoginSuccessLocation /btsync
        AuthUserFile "/etc/apache2/passwd"
        AuthName "My Login"
        Session On
        SessionMaxAge 1800
        SessionCookieName session path=/
        SessionCryptoPassphrase CHANGEME
        ErrorDocument 401 /login.html
        ProxyPass http://localhost:8888
        ProxyPassReverse http://localhost:8888
        RequestHeader set X-Forwarded-Proto "https"
        RequestHeader set X-Forwarded-Port "443"
    &lt;/Location&gt;
</code></pre>

<p>and add this under the &lt;VirtualHost *:443&gt; section</p>

<pre><code>        Redirect permanent /gui /btsync/gui
</code></pre>

<pre><code> sudo service btsync start

 sudo service apache2 restart

</code></pre>

<p>Now browse to <a href="https://yourdomain.com/btsync">https://yourdomain.com/btsync</a><br>
Click continue with out filling in a username or password. Accept the agreement and click continue. Click This is my first Sync 2.0 device, finally click Create Identity.</p>

<p>Click the Add Folder button:<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/btsync/addFolder.png"><br>
Browse to your watch directory. In our example it is /data/torrent/watch/. Then click the Open button.</p>

<p>Now click the 3 dots menu next to the folder and click Preferences:<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/btsync/FolderPreferences.png"><br>
Configure it how you want. This is how I have mine configured.</p>

<p>Now you can click the gear cog -&gt; My Devices to add your desktop and any mobile devices you want.<br>
When you download a torrent click the Open in BitTorrent Sync option and then the BitTorrent Sync app will launch.<br>
<img src="https://raw.githubusercontent.com/wiki/xombiemp/ultimate-torrent-setup/images/btsync/OpenIn.png"><br>
Just browse to the correct watch folder, based on the category of torrent, and then it will sync back to your Ubuntu server and be picked up and started immediately by ruTorrent and receive the correct label based on the folder you placed it in.</p>

    </div>

  </div>
  </div>
</div>
</div>
