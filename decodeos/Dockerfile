#
# Build this image with the command
#   docker build -f docker/build -t dyne/clojure:latest
#
# Then run with the command
#   docker run -p 3000:3000 -it dyne/clojure:latest
#

FROM dyne/devuan:beowulf
ENV debian buster

LABEL maintainer="Denis Roio <jaromil@dyne.org>" \
	  homepage="https://github.com/decodeproject/decode-os"

ENV LC_ALL C
ENV DEBIAN_FRONTEND noninteractive

ENV DYNESDK=https://sdk.dyne.org:4443/job \
	NETDATA_VERSION=1.10.0 \
	STEM_VERSION=1.6.0 \
	STEM_GIT=https://git.torproject.org/stem.git

ENV BUILD_DEPS="build-essential zlib1g-dev gcc make autoconf automake pkg-config uuid-dev golang"
WORKDIR /root

# Tor repository
RUN gpg --recv A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89 \
&& gpg --export A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89 | apt-key add -
RUN echo "deb https://deb.torproject.org/torproject.org $debian main" \
	>> /etc/apt/sources.list

# Nodejs repository
ADD https://deb.nodesource.com/gpgkey/nodesource.gpg.key nodesource.gpg.key
RUN apt-key add nodesource.gpg.key
RUN	echo "deb https://deb.nodesource.com/node_8.x $debian main" \
	>> /etc/apt/sources.list

RUN mkdir -p /usr/share/man/man1/ \
	&& apt-get -yy update && apt-get -yy upgrade \
	&& apt-get -yy install tor deb.torproject.org-keyring \
	   		   	   supervisor daemontools \
	   		   	   tmux curl redis-tools redis-server net-tools \
				   python3 python3-stem nodejs

RUN apt-get -yq install $BUILD_DEPS

# Latest Zenroom built static for x86-amd64 taken from our own builds at Dyne.org
ADD $DYNESDK/zenroom-static-amd64/lastSuccessfulBuild/artifact/src/zenroom-static /usr/bin/zenroom
RUN chmod +x /usr/bin/zenroom

# Compile some software from the source
WORKDIR /usr/src

# Stem built from source
# RUN git clone $STEM_GIT && cd stem && git checkout -b $STEM_VERSION $STEM_VERSION && python3 setup.py install

# Configure Tor Controlport auth
ENV	TORDAM_GIT=github.com/decodeproject/tor-dam
RUN torpass=`echo "print(RNG.new():octet(16):base58())" | zenroom` \
	&& go get -v -u $TORDAM_GIT/... && cd ~/go/src/github.com/decodeproject/tor-dam \
	&& sed -i python/damhs.py -e "s/topkek/$torpass/" \
	&& sed -i python/damauth.py -e "s/topkek/$torpass/" \
	&& make install && make -C contrib install-init \
	&& torpasshash=`HOME=/var/lib/tor setuidgid debian-tor tor --hash-password "$torpass"` \
	&& sed -e 's/User tor/User debian-tor/' < contrib/torrc > /etc/tor/torrc \
	&& sed -e 's/HashedControlPassword .*//' -i /etc/tor/torrc \
	&& echo "HashedControlPassword $torpasshash" >> /etc/tor/torrc
RUN chmod -R go-rwx /etc/tor && chown -R debian-tor /etc/tor \
	&& rm -rf /var/lib/tor/data && chown -R debian-tor /var/lib/tor \
	&& mkdir -p /var/run/tor && chown -R debian-tor /var/run/tor
RUN cp /root/go/bin/dam* /usr/bin

# fix npm - not the latest version installed by apt-get
RUN npm install -g npm
RUN npm install -g redis-commander
ENV REDIS_HOSTS=localhost

# Netdata
ADD https://github.com/firehol/netdata/releases/download/v$NETDATA_VERSION/netdata-${NETDATA_VERSION}.tar.gz netdata.tgz
RUN tar xf netdata.tgz && cd netdata-$NETDATA_VERSION \
	&& ./netdata-installer.sh --dont-wait --dont-start-it \
	&& cd - && rm -rf netdata.tgz netdata-$NETDATA_VERSION

# Openresty
ADD https://openresty.org/package/pubkey.gpg openresty.gpg
RUN apt-key add openresty.gpg
RUN echo "deb http://openresty.org/package/debian stretch openresty" \
>> /etc/apt/sources.list
RUN apt-get -yq update \
&& apt-get -yq install --no-install-recommends openresty

# cleanup
RUN apt-get -yq remove --purge $BUILD_DEPS \
&& apt-get -yq --purge autoremove && apt-get -yq clean \
&& npm cache clean --force && npm uninstall -g npm

COPY supervisord.conf /etc/supervisor/supervisord.conf

RUN groupadd -g 6000 app && useradd -r -u 6000 -g app -d /home/app app
WORKDIR /home/app
RUN chown -R app:app /home/app

# Tor's socks5
EXPOSE 9150
# supervisor
EXPOSE 9001 9001
# redis-commander
EXPOSE 8081 8081
# netdata
EXPOSE 19999 19999

CMD /etc/init.d/supervisor start
