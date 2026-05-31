FROM node:latest
COPY . /hello-mars
RUN cd /hello-mars; npm install
EXPOSE 3000
WORKDIR /hello-mars 
ENTRYPOINT ["npm", "start"]