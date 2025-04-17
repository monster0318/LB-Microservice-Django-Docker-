# Load Balancer Microservices Django-Docker

📘 [English Version - README.en.md](https://github.com/twtrubiks/docker-django-nginx-uwsgi-postgres-tutorial/blob/master/README.en.md)

Basic tutorial on Docker + Django + Nginx + uWSGI + PostgreSQL – From Zero to Deployment.

Learn how to build a production-ready stack using [Docker](https://www.docker.com/) with [Django](https://www.djangoproject.com/), [Nginx](https://nginx.org/en/), [uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/), and [PostgreSQL](https://www.postgresql.org/) 📝

If you're new to Docker, I recommend starting with this beginner-friendly guide:

👉 [Docker Beginners Guide – Build Django + PostgreSQL using Docker 📝](https://github.com/twtrubiks/docker-tutorial)

### YouTube Video Tutorials:
- [PART 1 – Overview](https://youtu.be/u4XIMTOsxJk)
- [PART 2 – Architecture & Steps](https://youtu.be/9K4O1UuaXrU)
- [PART 3 – Hands-on Project](https://youtu.be/v7Mf9TuROnc)
- [Nginx Log Analysis: PV & UV](https://youtu.be/mUyDVVX6OD4) – [Quick Article Link](https://github.com/twtrubiks/docker-django-nginx-uwsgi-postgres-tutorial#%E9%80%8F%E9%81%8E-nginx-log-%E5%88%86%E6%9E%90-pv-uv)
- [Nginx Auth Basic Tutorial](https://youtu.be/zWODI3YHb2Y) – [Quick Article Link](https://github.com/twtrubiks/docker-django-nginx-uwsgi-postgres-tutorial#%E8%A8%AD%E5%AE%9A-auth_basic)

---

## Introduction

### Docker
![](https://i.imgur.com/gDcSwcs.png)

Already covered in my [Docker Beginners Guide](https://github.com/twtrubiks/docker-tutorial), so we won’t go over it again here 😛

---

### Django
Reference guides:
- [Django Beginners Guide 📝](https://github.com/twtrubiks/django-tutorial)
- [Django REST Framework Beginners Guide 📝](https://github.com/twtrubiks/django-rest-framework-tutorial)

For more Django examples, visit my [GitHub repositories](https://github.com/twtrubiks?utf8=%E2%9C%93&tab=repositories&q=Django)

---

### PostgreSQL
![](https://i.imgur.com/RrNtbfz.png)

---

### Nginx
![](https://i.imgur.com/AkcCtDa.png)

Nginx is a lightweight, high-performance web server known for its efficiency and stability. One of the key things it solves is the **C10K problem** (supporting 10,000+ concurrent clients). More on that here: [The C10K problem](http://www.kegel.com/c10k.html)

But Nginx can't handle dynamic content (like Python). That's where [uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/) comes in. Here’s the architecture:

🌐 Web Client → 🌐 Nginx → 🔗 Unix Socket → 🔌 uWSGI → 🐍 Django

**Why uWSGI?**
uWSGI acts as a bridge between Nginx and Django, communicating using the uWSGI protocol.

**Why still use Nginx?**
- Nginx serves static content (HTML, CSS, JS, images) efficiently.
- Better performance than uWSGI for static files.
- Supports caching, reverse proxying, and load balancing.

💡 *Tip:* To learn more about reverse proxying, check out this explanation: [Forward Proxy vs Reverse Proxy](https://github.com/twtrubiks/docker-django-nginx-uwsgi-postgres-load-balance-tutorial#%E6%AD%A3%E5%90%91%E4%BB%A3%E7%90%86%E5%99%A8--vs-%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86%E5%99%A8)

---

### Why do I not need Nginx/uWSGI during development?

When developing locally with `python manage.py runserver`, Django runs its own lightweight server. It’s fine for development but not suitable for production due to performance and security limitations.

---

### Why use uWSGI over Gunicorn?

In Heroku, Gunicorn is recommended, and I’ve used it in these tutorials:
- [Deploying Django to Heroku](https://github.com/twtrubiks/Deploying_Django_To_Heroku_Tutorial)
- [Deploying Flask to Heroku](https://github.com/twtrubiks/Deploying-Flask-To-Heroku)

Both uWSGI and Gunicorn are great—choose based on your specific deployment needs.

---

### What about Apache?

Both [Nginx](https://nginx.org/en/) and [Apache](https://httpd.apache.org/) are excellent. It depends on your requirements and environment. There’s no single "best" server—choose what works best for your case 👍

---

## Tutorial Overview

We'll use Docker to create three containers:
- Nginx
- Django + uWSGI
- PostgreSQL

This is based on [uWSGI + Django + Nginx tutorial](https://uwsgi-docs.readthedocs.io/en/latest/tutorials/Django_and_nginx.html), with some custom tweaks.

---

### Nginx Setup

Check out the [`nginx/Dockerfile`](https://github.com/twtrubiks/docker-django-nginx-uswgi-postgres-tutorial/blob/master/nginx/Dockerfile):

```Dockerfile
FROM nginx:latest

COPY nginx.conf /etc/nginx/nginx.conf
COPY my_nginx.conf /etc/nginx/sites-available/

RUN mkdir -p /etc/nginx/sites-enabled/ \
    && ln -s /etc/nginx/sites-available/my_nginx.conf /etc/nginx/sites-enabled/

CMD ["nginx", "-g", "daemon off;"]
```

**Explanation:**

1. Copy `nginx.conf` to `/etc/nginx/nginx.conf`.  
   You can grab the original config from a running nginx container at `/etc/nginx/nginx.conf`.  
   See the original: [`nginx_origin.conf`](https://github.com/twtrubiks/docker-django-nginx-uswgi-postgres-tutorial/blob/master/nginx/nginx_origin.conf)

2. Modify:
   ```conf
   user root;
   ```
   And:
   ```conf
   # include /etc/nginx/conf.d/*.conf;
   include /etc/nginx/sites-available/*;
   ```

   This disables the default config and enables our custom site config.

3. Copy `my_nginx.conf` to `/etc/nginx/sites-available/`.

   Note: If you're using `nginx:latest`, the directories `/etc/nginx/sites-available/` and `/etc/nginx/sites-enabled/` may not exist. You can create them yourself (see Dockerfile commands).

4. Use a symbolic link to enable the site:
   ```sh
   ln -s /etc/nginx/sites-available/my_nginx.conf /etc/nginx/sites-enabled/
   ```

---

### `my_nginx.conf` Example

```nginx
upstream uwsgi {
    server unix:/docker_api/app.sock;
}

server {
    listen 80;
    server_name twtrubiks.com www.twtrubiks.com;
    charset utf-8;

    client_max_body_size 75M;

    location /static {
        alias /docker_api/static;
    }

    location / {
        uwsgi_pass uwsgi;
        include /etc/nginx/uwsgi_params;
    }
}
```

---
