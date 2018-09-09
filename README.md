#
# instal Elicoin core + iquidus explorer - Debian 9
# usage software:
# putty - remote terminal
# https://www.putty.org/
#
# winscp - simple create,edit folder/file
# https://winscp.net/
#
# pspad - formatted text editor with ftp client (good default editor for winscp)
# http://www.pspad.com/cz/
#
# important: example is installed under root, if you are installing under another user, use sudo and change user and paths!
#

apt update && apt upgrade -y
apt install build-essential libtool autotools-dev automake autoconf pkg-config libssl-dev libboost-all-dev
apt install libqrencode-dev libminiupnpc-dev libevent-dev libcap-dev libseccomp-dev git bsdmainutils
apt install software-properties-common libqt5gui5 libqt5core5a libqt5dbus5 qttools5-dev qttools5-dev-tools libprotobuf-dev protobuf-compiler

nano /etc/apt/sources.list.d/deeponionAdd.list
#copy these two lines and Add into (right click to terminal window):
       deb http://ppa.launchpad.net/bitcoin/bitcoin/ubuntu xenial main
       deb http://ftp.de.debian.org/debian jessie-backports main

mkdir /root/Downloads
# Now go to Bitcoin repository key (ref) copy the text( start and end with ----- ), paste in the file BTC.gpg in ~/Downloads
# https://keyserver.ubuntu.com/pks/lookup?op=get&search=0xD46F45428842CE5E
nano /root/Downloads/BTC.gpg
apt-key add ~/Downloads/BTC.gpg
# terminal: OK

apt update
apt install libdb4.8-dev libdb4.8++-dev


git clone https://github.com/elicoin/elicoin.git
cd elicoin
./autogen.sh
./configure
# ! is long time
make
make install

# is here /usr/local/bin/elicoind

# start daemon
elicoind  -daemon -txindex

# stop daemon
elicoin-cli stop

# go to the /root/.elicoin/ folder and create the elicoin.conf file
cd /root/.elicoin/
touch elicoin.conf

#and write there:  (other optional options according to the manual)

    rpcuser=<your_hoice_of_username>
    rpcpassword=<your_hoice_of_password>
    rpcport=9999
    rpcthreads=8
    rpcallowip=127.0.0.1
    maxconnections=12
    gen=0
    server=1
    daemon=1

# save the file
# start daemon (synchronization takes several hours, according to hw)
elicoind  -daemon -txindex

# install npm (require: nodejs, mongo)
apt install curl

# nodejs
curl -sL https://deb.nodesource.com/setup_8.x | bash -
apt install nodejs
apt install gcc g++ make

curl -sL https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list
apt update && apt install yarn

node --version
npm --version
yarn --version

# mongo ! ignore the alert message
apt-key adv --keyserver keyserver.ubuntu.com --recv 7F0CEB10
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 9DA31620334BD75D9DCB49F368818C72E52529D4
echo "deb http://repo.mongodb.org/apt/debian stretch/mongodb-org/4.0 main" | tee /etc/apt/sources.list.d/mongodb-org-4.0.list
apt update
# not use -y ! but then enter yes
apt install  mongodb-org

# fix - will not be updated
echo "mongodb-org hold" | dpkg --set-selections
echo "mongodb-org-server hold" | dpkg --set-selections
echo "mongodb-org-shell hold" | dpkg --set-selections
echo "mongodb-org-mongos hold" | dpkg --set-selections
echo "mongodb-org-tools hold" | dpkg --set-selections

# start service
service mongod start

# start mongo
mongo

> use explorerdb

#Create user with read/write access:
> db.createUser( { user: "iquidus", pwd: "3xp!0reR", roles: [ "readWrite" ] } )

> exit

# set autostart on boot
systemctl enable mongod.service

# npm
git clone https://github.com/iquidus/explorer explorer
cd explorer && npm install --production
cp ./settings.json.template ./settings.json
nano settings.json

#edit // wallet settings - "<user>", "<pass>" like rpcuser, rpcpassword

apt install screen -y
# start screen
screen
#(press spacebar)

npm start
npm stop

# *) edit /root/explorer/lib/explorer.js Line 341 (use winscp)
# replace:
      module.exports.get_rawtransaction(input.txid, function(tx){
        if (tx) {
          module.exports.syncLoop(tx.vout.length, function (loop) {
            var i = loop.iteration();
            if (tx.vout[i].n == input.vout) {
              //module.exports.convert_to_satoshi(parseFloat(tx.vout[i].value), function(amount_sat){
              if (tx.vout[i].scriptPubKey.addresses) {
                addresses.push({hash: tx.vout[i].scriptPubKey.addresses[0], amount:tx.vout[i].value});
              }
                loop.break(true);
                loop.next();
              //});
            } else {
              loop.next();
            }
          }, function(){
            return cb(addresses);
          });
        } else {
          return cb();
        }
});
#with this:
  module.exports.get_rawtransaction(input.txid, function(tx){
    if (tx) {
      if (tx.vout) { //Added
        console.log('tx.vout: %o', tx.vout); //Added
        module.exports.syncLoop(tx.vout.length, function (loop) {
          var i = loop.iteration();
          if (tx.vout[i].n == input.vout) {
            //module.exports.convert_to_satoshi(parseFloat(tx.vout[i].value), function(amount_sat){
            if (tx.vout[i].scriptPubKey.addresses) {
              addresses.push({hash: tx.vout[i].scriptPubKey.addresses[0], amount:tx.vout[i].value});
            }
              loop.break(true);
              loop.next();
            //});
          } else {
            loop.next();
          }
        }, function(){
          return cb(addresses);
        });
      } else {
        return cb();
      }
    } else {//Added
      return cb();//Added
    }//Added
  });

# save

npm start

#(press Ctrl+a, Ctrl+d) quit this screen

# start other screen
screen
#(press spacebar)

# initial manual db synchronization   (takes several hours, according to hw)
/usr/bin/nodejs scripts/sync.js index update

# if alert "script is already running"
rm tmp/index.pid

# crontab - db update data every minute (set up after db synchronization)
crontab -e
*/1 * * * * cd /<path>/<to>/explorer && /usr/bin/nodejs scripts/sync.js index update > /dev/null 2>&1

# after initial manual db synchronization, it is possible to delete the screen for it

# install watch dog for explorer
npm install forever -g
install forever-monitor
npm install forever-monitor
forever start bin/cluster

# set autostart elicoind on boot - requirement file "elicoin"  in /etc/init.d/
chmod +x "/etc/init.d/elicoin"
touch "/var/log/elicoin.log"
#chown "root" "/var/log/elicoin.log"
update-rc.d "elicoin" defaults

# manage daemon use script
# daemon - start|stop|restart|uninstall
/etc/init.d/elicoin start

# set autostart npm on boot - requirement file "explorer"  in /etc/init.d/
chmod +x "/etc/init.d/explorer"
touch "/var/log/explorer.log"
#chown "root" "/var/log/explorer.log"
update-rc.d "explorer" defaults

# manage daemon use script
# daemon - start|stop|restart|uninstall
/etc/init.d/explorer start


# end

-------------------------------
COMMANDS:
# list screen
screen -ls
# open screen
screen -r your_session
# quit screen
screen -X -S your_session_with_elicoind quit

# start daemon
elicoind  -daemon -txindex

# stop daemon
elicoin-cli stop

# reindex (if not corect stop)
elicoind -reindex-chainstate  -daemon -txindex

# mongo
service mongod start
service mongod stop

# npm
npm start
npm stop

-------------------------------
SOURCES:
# install Elicoin core
http://www.bubasik.com/yiimp-pool-install-ubuntu-guide-16-04-server-and-setup-elicoin/
# nodejs.org
https://nodejs.org/en/
# mongo
https://docs.mongodb.com/manual/tutorial/install-mongodb-on-debian/
# iquidus explorer
https://github.com/iquidus/explorer
# error*:
https://github.com/iquidus/explorer/issues/118
# Public Key Server -- Get "0xd46f45428842ce5e "
https://keyserver.ubuntu.com/pks/lookup?op=get&search=0xD46F45428842CE5E
# libdb4.8
https://github.com/deeponion/deeponion/blob/master/doc/build-debian-9.md
# start script
https://gist.github.com/naholyr/4275302

-------------------------------
*)
/explorer/lib/explorer.js:344
module.exports.syncLoop(tx.vout.length, function (loop) {
^
TypeError: Cannot read property 'length' of undefined
at /root/explorer/lib/explorer.js:344:42
at Request._callback (/root/explorer/lib/explorer.js:114:14)
at Request.self.callback (/root/explorer/node_modules/request/request.js:187:22)
at emitTwo (events.js:125:13)
at Request.emit (events.js:213:7)
at Request. (/root/explorer/node_modules/request/request.js:1044:10)
at emitOne (events.js:115:13)
at Request.emit (events.js:210:7)
at IncomingMessage. (/root/explorer/node_modules/request/request.js:965:12)
at emitNone (events.js:110:20)
at IncomingMessage.emit (events.js:207:7)
at endReadableNT (_stream_readable.js:1059:12)
at _combinedTickCallback (internal/process/next_tick.js:138:11)
at process._tickCallback (internal/process/next_tick.js:180:9)

# more
my help, your help - www search...
