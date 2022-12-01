FROM debian

RUN apt update -y
RUN apt install git -y
RUN apt install python3 -y

RUN /bin/mkdir /srv/server
RUN git -C /srv/server clone https://github.com/GammaRay99/web_server.git
CMD ["python3", "/srv/server/web_server/example.py"]