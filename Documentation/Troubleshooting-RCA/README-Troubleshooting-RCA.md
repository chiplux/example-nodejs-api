# Troubleshooting & RCA

## Problems, Cause, and Solution

After some checking on the servers, we found that the application is not running. Problem that we found:

### **Problem**: Could not execute docker commands, although we already add `ubuntu` user to `docker` group.

**Identify**: There are `alias` in `/home/ubuntu/.bashrc` that override docker commands.

**Solution**: Disable/remove the `alias` in `/home/ubuntu/.bashrc`.

### **Problem**: Docker daemon is not running.
**Identify**: Checked `systemctl status docker` and found that the docker daemon is not running. use `sudo journalctl -xeu docker.service` to check the error logs. There is possible cause because of the wrong docker configuration.
```
$ cat /etc/docker/daemon.json
{   "userns-remap-to": "default" }
```
**Solution**: Fix the configuration in /etc/docker/daemon.json from `{   "userns-remap-to": "default" }` to `{   "userns-remap": "default" }` and restart the docker daemon.

### **Problem**: Could not connect to the database.
**Identify**: Checked `docker compose logs db` and found there are logs like:
```
ECONNREFUSED ::1:5432
ECONNREFUSED 127.0.0.1:5432
```
**Solution**: Modify the DB_HOST from `localhost` to `db` in the `docker-compose.yml` file and rerun command:
```
$ docker compose up -d --build
```

### **Problem**: Could not run service in port 3000.
**Identify**: Check the port 3000 and service that run in it using command:
```
$ sudo netstat -ntlp | grep :3000
tcp        0      0 0.0.0.0:3000            0.0.0.0:*               LISTEN      597/nginx: master p
ubuntu@ip-10-0-6-115:~/example-nodejs-api$ sudo lsof -i :3000
COMMAND PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx   597     root    5u  IPv4   6090      0t0  TCP *:3000 (LISTEN)
nginx   599 www-data    5u  IPv4   6090      0t0  TCP *:3000 (LISTEN)
nginx   605 www-data    5u  IPv4   6090      0t0  TCP *:3000 (LISTEN)
```
Found that the service that run in port 3000 is `nginx`.
**Solution**: Stop the `nginx` service and change the nginx port to 80, commands:
```
ubuntu@ip-10-0-6-115:~/example-nodejs-api$ grep -R "3000" /etc/nginx
/etc/nginx/sites-available/default:	listen 3000 default_server;
/etc/nginx/sites-enabled/default:	listen 3000 default_server;
ubuntu@ip-10-0-6-115:~/example-nodejs-api$ sudo nano /etc/nginx/sites-available/default
ubuntu@ip-10-0-6-115:~/example-nodejs-api$ grep -R "3000" /etc/nginx
ubuntu@ip-10-0-6-115:~/example-nodejs-api$ sudo systemctl start nginx
ubuntu@ip-10-0-6-115:~/example-nodejs-api$ sudo systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-02-15 19:41:42 UTC; 12s ago
       Docs: man:nginx(8)
    Process: 4364 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 4365 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 4367 (nginx)
      Tasks: 3 (limit: 2067)
     Memory: 2.3M (peak: 2.5M)
        CPU: 20ms
     CGroup: /system.slice/nginx.service
             ├─4367 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─4368 "nginx: worker process"
             └─4369 "nginx: worker process"

Feb 15 19:41:42 ip-10-0-6-115 systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
Feb 15 19:41:42 ip-10-0-6-115 systemd[1]: Started nginx.service - A high performance web server and a reverse proxy server.
```

### **Problem**: Could not access the application on port 3000 in the server.
**Identify**: Checked using this commands
```
ubuntu@ip-10-0-6-115:~/example-nodejs-api$ docker compose ps
NAME                       IMAGE                      COMMAND                  SERVICE   CREATED         STATUS          PORTS
example-nodejs-api-api-1   example-nodejs-api-api     "docker-entrypoint.s…"   api       8 minutes ago   Up 16 seconds
example-nodejs-api-db-1    postgres:18.1-alpine3.23   "docker-entrypoint.s…"   db        8 minutes ago   Up 8 minutes    0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp
ubuntu@ip-10-0-6-115:~/example-nodejs-api$ npx autocannon -c 10 -d 5 --renderStatusCodes http://localhost:3000/api/cities
Running 5s test @ http://localhost:3000/api/cities
10 connections


┌─────────┬──────┬──────┬───────┬──────┬──────┬───────┬──────┐
│ Stat    │ 2.5% │ 50%  │ 97.5% │ 99%  │ Avg  │ Stdev │ Max  │
├─────────┼──────┼──────┼───────┼──────┼──────┼───────┼──────┤
│ Latency │ 0 ms │ 0 ms │ 0 ms  │ 0 ms │ 0 ms │ 0 ms  │ 0 ms │
└─────────┴──────┴──────┴───────┴──────┴──────┴───────┴──────┘
┌───────────┬─────┬──────┬─────┬───────┬─────┬───────┬─────┐
│ Stat      │ 1%  │ 2.5% │ 50% │ 97.5% │ Avg │ Stdev │ Min │
├───────────┼─────┼──────┼─────┼───────┼─────┼───────┼─────┤
│ Req/Sec   │ 0   │ 0    │ 0   │ 0     │ 0   │ 0     │ 0   │
├───────────┼─────┼──────┼─────┼───────┼─────┼───────┼─────┤
│ Bytes/Sec │ 0 B │ 0 B  │ 0 B │ 0 B   │ 0 B │ 0 B   │ 0 B │
└───────────┴─────┴──────┴─────┴───────┴─────┴───────┴─────┘
┌──────┬───────┐
│ Code │ Count │
└──────┴───────┘

Req/Bytes counts sampled once per second.
# of samples: 5

23k requests in 5.03s, 0 B read
23k errors (0 timeouts)
npm notice
npm notice New minor version of npm available! 11.6.2 -> 11.10.0
npm notice Changelog: https://github.com/npm/cli/releases/tag/v11.10.0
npm notice To update run: npm install -g npm@11.10.0
npm notice
ubuntu@ip-10-0-6-115:~/example-nodejs-api$ curl http://localhost:3000/api/cities
curl: (7) Failed to connect to localhost port 3000 after 0 ms: Couldn't connect to server
```
**Solution**: From the docker container, we can see that port 3000 is not exposed to the host. We need to check port configuration, in `docker-compose.yml` file. it should use quote symbol `"`, from `ports: - 3000:3000` changed to `ports: - "3000:3000"` for port mapping. Then run `docker compose up -d --build` to update the container. Commands:
```
ubuntu@ip-10-0-6-115:~/example-nodejs-api$ docker compose ps
NAME                       IMAGE                      COMMAND                  SERVICE   CREATED         STATUS         PORTS
example-nodejs-api-api-1   example-nodejs-api-api     "docker-entrypoint.s…"   api       5 seconds ago   Up 5 seconds   0.0.0.0:3000->3000/tcp, [::]:3000->3000/tcp
example-nodejs-api-db-1    postgres:18.1-alpine3.23   "docker-entrypoint.s…"   db        5 seconds ago   Up 5 seconds   0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp
ubuntu@ip-10-0-6-115:~/example-nodejs-api$ curl http://localhost:3000/api/cities
[{"id":1,"state":"California","country":"USA","city_name":"Los Angeles"},{"id":2,"state":"New York","country":"USA",
```

### **Problem**: There were high request error, command:
```
ubuntu@ip-10-0-6-115:~/example-nodejs-api$ npx autocannon -c 10 -d 5 --renderStatusCodes http://localhost:3000/api/cities
Running 5s test @ http://localhost:3000/api/cities
10 connections


┌─────────┬────────┬────────┬────────┬────────┬───────────┬─────────┬────────┐
│ Stat    │ 2.5%   │ 50%    │ 97.5%  │ 99%    │ Avg       │ Stdev   │ Max    │
├─────────┼────────┼────────┼────────┼────────┼───────────┼─────────┼────────┤
│ Latency │ 100 ms │ 102 ms │ 112 ms │ 136 ms │ 103.32 ms │ 5.06 ms │ 136 ms │
└─────────┴────────┴────────┴────────┴────────┴───────────┴─────────┴────────┘
┌───────────┬───────┬───────┬─────────┬─────────┬───────┬─────────┬───────┐
│ Stat      │ 1%    │ 2.5%  │ 50%     │ 97.5%   │ Avg   │ Stdev   │ Min   │
├───────────┼───────┼───────┼─────────┼─────────┼───────┼─────────┼───────┤
│ Req/Sec   │ 90    │ 90    │ 100     │ 100     │ 96    │ 4.9     │ 90    │
├───────────┼───────┼───────┼─────────┼─────────┼───────┼─────────┼───────┤
│ Bytes/Sec │ 31 kB │ 31 kB │ 34.4 kB │ 34.4 kB │ 33 kB │ 1.69 kB │ 31 kB │
└───────────┴───────┴───────┴─────────┴─────────┴───────┴─────────┴───────┘
┌──────┬───────┐
│ Code │ Count │
├──────┼───────┤
│ 500  │ 480   │
└──────┴───────┘

Req/Bytes counts sampled once per second.
# of samples: 5

0 2xx responses, 480 non 2xx responses
490 requests in 5.03s, 165 kB read
```
**Identify**: From the result, we can see there are 480 non 2xx responses, which means there are 480 error responses. We check the logs to see the error messages. Commands:
```
ubuntu@ip-10-0-6-115:~/example-nodejs-api$ docker compose logs api --tail=100
api-1  |     at async /app/app.js:59:20
api-1  | {"timestamp":"2026-02-15T19:48:16.653Z","method":"GET","url":"/api/cities","clientIp":"::ffff:172.18.0.1","host":"localhost:3000","status":500,"durationMs":102.467}
api-1  | Error fetching data from /api/cities: Error: timeout exceeded when trying to connect
api-1  |     at /app/node_modules/pg-pool/index.js:45:11
api-1  |     at async /app/app.js:59:20
...
```
**Solution**: We need to check the configuration related to the connection pool of the database connection in `app.js` file and tune up the parameters. Configuration changes:
```
...
const pool = new Pool({
  user: process.env.DB_USER,
  host: process.env.DB_HOST,
  database: process.env.DB_NAME,
  password: process.env.DB_PASSWORD,
  port: process.env.DB_PORT,

  // ⭐ tuned for small service
  max: 10,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
});
...
```
and
```
app.get('/api/cities', async (req, res) => {
  const client = await pool.connect();
  try {
    const result = await client.query('SELECT * FROM cities');
    res.status(200).json(result.rows);
  } catch (err) {
    console.error('Error fetching data from /api/cities:', err);
    res.status(500).json({ message: 'Error fetching cities data', error: err.message });
  } finally {
    client.release(); // ⭐⭐⭐ CRITICAL
  }
});
```
Rebuild and test it again.
```
ubuntu@ip-10-0-6-115:~/example-nodejs-api$ docker compose down
docker compose up --build -d
[+] down 3/3
 ✔ Container example-nodejs-api-api-1 Removed                                                                                                                                                                0.7s
 ✔ Container example-nodejs-api-db-1  Removed                                                                                                                                                                0.2s
 ✔ Network example-nodejs-api_default Removed                                                                                                                                                                0.1s
[+] Building 0.6s (12/12) FINISHED
 => [internal] load local bake definitions                                                                                                                                                                  0.0s
 => => reading from stdin 524B                                                                                                                                                                              0.0s
 => [internal] load build definition from Dockerfile                                                                                                                                                        0.0s
 => => transferring dockerfile: 153B                                                                                                                                                                        0.0s
 => [internal] load metadata for docker.io/library/node:25-alpine3.22                                                                                                                                       0.2s
 => [internal] load .dockerignore                                                                                                                                                                           0.0s
 => => transferring context: 2B                                                                                                                                                                             0.0s
 => [1/5] FROM docker.io/library/node:25-alpine3.22@sha256:be0c5c5272d451d5faddfa4654495c530c51ab437b0d901a9c533d9635ad832a                                                                                 0.0s
 => [internal] load build context                                                                                                                                                                           0.0s
 => => transferring context: 4.58kB                                                                                                                                                                         0.0s
 => CACHED [2/5] WORKDIR /app                                                                                                                                                                               0.0s
 => CACHED [3/5] COPY package*.json ./                                                                                                                                                                      0.0s
 => CACHED [4/5] RUN npm install                                                                                                                                                                            0.0s
 => [5/5] COPY . .                                                                                                                                                                                          0.1s
 => exporting to image                                                                                                                                                                                      0.1s
 => => exporting layers                                                                                                                                                                                     0.0s
 => => writing image sha256:1c626ca94d2db362ed3f13900a0cd84f10ea78bea079b67443ad5a2da7b79dce                                                                                                                0.0s
 => => naming to docker.io/library/example-nodejs-api-api                                                                                                                                                   0.0s
 => resolving provenance for metadata file                                                                                                                                                                  0.0s
[+] up 4/4
 ✔ Image example-nodejs-api-api       Built                                                                                                                                                                  0.7s
 ✔ Network example-nodejs-api_default Created                                                                                                                                                                0.1s
 ✔ Container example-nodejs-api-db-1  Created                                                                                                                                                                0.1s
 ✔ Container example-nodejs-api-api-1 Created                                                                                                                                                                0.0s
ubuntu@ip-10-0-6-115:~/example-nodejs-api$ npx autocannon -c 10 -d 5 --renderStatusCodes http://localhost:3000/api/cities
Running 5s test @ http://localhost:3000/api/cities
10 connections


┌─────────┬────────┬────────┬────────┬────────┬──────────┬──────────┬────────┐
│ Stat    │ 2.5%   │ 50%    │ 97.5%  │ 99%    │ Avg      │ Stdev    │ Max    │
├─────────┼────────┼────────┼────────┼────────┼──────────┼──────────┼────────┤
│ Latency │ 233 ms │ 253 ms │ 266 ms │ 266 ms │ 251.8 ms │ 10.22 ms │ 266 ms │
└─────────┴────────┴────────┴────────┴────────┴──────────┴──────────┴────────┘
┌───────────┬─────┬──────┬─────┬─────────┬─────────┬─────────┬─────────┐
│ Stat      │ 1%  │ 2.5% │ 50% │ 97.5%   │ Avg     │ Stdev   │ Min     │
├───────────┼─────┼──────┼─────┼─────────┼─────────┼─────────┼─────────┤
│ Req/Sec   │ 0   │ 0    │ 0   │ 10      │ 2       │ 4       │ 10      │
├───────────┼─────┼──────┼─────┼─────────┼─────────┼─────────┼─────────┤
│ Bytes/Sec │ 0 B │ 0 B  │ 0 B │ 9.37 kB │ 1.87 kB │ 3.75 kB │ 9.36 kB │
└───────────┴─────┴──────┴─────┴─────────┴─────────┴─────────┴─────────┘
┌──────┬───────┐
│ Code │ Count │
├──────┼───────┤
│ 200  │ 10    │
└──────┴───────┘

Req/Bytes counts sampled once per second.
# of samples: 5

20 requests in 5.04s, 9.36 kB read
```
The application is running and the test successfully.

## Note
We can check the file log-troubleshoot.txt for more information.