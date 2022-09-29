# johanv.net
my personal website

For more info on the process and reasoning behind it, see [this blog post](https://johanv.net/blog/how-i-created-johanv-net/).

# main setup

```bash
$ cd ~/git/johanvandegriff/hugowncast/
$ docker build . -t johanvandegriff/hugowncast:build1
$ docker login -u johanvandegriff #paste in key from hub.docker.com
$ docker push johanvandegriff/hugowncast:build1

$ cd ~/git/johanvandegriff/multistream
$ docker build . -t johanvandegriff/multistream:build1
$ docker push johanvandegriff/multistream:build1

$ cat <<EOF >> ~/.ssh/config

Host johanv.net
	HostName johanv.net
	User rancher
	IdentityFile ~/.ssh/digitalocean
EOF

$ ssh johanv.net
[rancher@johanv ~]$ docker network create johanvnet
[rancher@johanv ~]$ docker run --name www -d --restart unless-stopped --net johanvnet -v ~/owncast-data:/app/data -v ~/hugo:/app/hugo -it johanvandegriff/hugowncast:build11
[rancher@johanv ~]$ docker run --name caddy -d --restart unless-stopped --net johanvnet -p 80:80 -p 443:443 -v ~/Caddyfile:/etc/caddy/Caddyfile -v ~/caddy-data:/data caddy
[rancher@johanv ~]$ docker run --name rtmp -d --restart unless-stopped --net johanvnet -p 1935:1935 -v ~/rtmp:/srv johanvandegriff/multistream:build2
[rancher@johanv ~]$ sudo chown rancher:rancher -R ~
[rancher@johanv ~]$ exit

$ mkdir ~/sshfs/johanv.net
$ sshfs johanv.net: ~/sshfs/johanv.net/

# now we can edit the site locally :]
$ cd ~/sshfs/johanv.net/hugo
$ hugo new blog/post4.md
$ nano content/blog/post4.md

$ cd ~/sshfs/johanv.net/chat-fwd
$ nano secret-dlive.env secret-nginx.conf secret-owncast.env secret-twitch.env secret-youtube.env
#or copy them from the old site
#make sure secret-nginx.conf is empty, because we are not using this for rtmp, just chat fwd
$ ssh johanv.net docker restart chat-fwd

$ cd ~/sshfs/johanv.net
$ cat <<EOF >> Caddyfile
{
	email user@example.com
}

johanv.net {
        reverse_proxy www:8080
}

www.johanv.net {
        redir https://johanv.net{uri} html
}

rtmp.johanv.net {
        reverse_proxy chat-fwd:8080
}
EOF
$ ssh johanv.net docker restart caddy
$ docker stop www
$ rsync -aP --itemize-changes --delete ~/git/johanvandegriff/johanv.net/ johanv.net:hugo/
$ docker start www
```

# more stuff
```bash
docker run --name m2m2m -d --restart unless-stopped --net johanvnet johanvandegriff/m2m2m:build1
docker run --name games -d --restart unless-stopped --net johanvnet -v ~/games:/srv johanvandegriff/games:build7
docker run --name games-mongodb -d --restart unless-stopped --net johanvnet -v ~/games-mongodb:/data/db mongo
docker run --name asciiradio -d --restart unless-stopped --net johanvnet -p 1337:1337 -p 1338:1338 johanvandegriff/asciiradio:build2
```

Caddyfile:
```
m2m2m.johanv.net {
	reverse_proxy m2m2m:80
}
games.johanv.net {
	reverse_proxy games:5000
}
asciiradio.johanv.net {
	reverse_proxy asciiradio:8080
}
```

# Importing MongoDB Data
```
echo -n 'mongodb://games-mongodb:27017/boggle' > games/boggle/mongodb-url.txt

scp backup/boggle-mongodb-backup.json johanv.net:
ssh johanv.net
sudo mv boggle-mongodb-backup.json games-mongodb/
docker exec -it games-mongodb mongoimport --db boggle --collection games --file /data/db/boggle-mongodb-backup.json --jsonArray
sudo rm games-mongodb/boggle-mongodb-backup.json
```

# File upload service
```bash
docker run --name files -d --restart unless-stopped --net johanvnet -v ~/files:/tmp docker.io/svenstaro/miniserve /tmp
pw=$(echo -n "password" | sha256sum | cut -f 1 -d ' ')
docker run --name upload -d --restart unless-stopped --net johanvnet -v ~/files:/tmp docker.io/svenstaro/miniserve /tmp --upload-files --overwrite-files --auth johanv:sha256:$pw
```
