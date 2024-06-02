FROM node:10-alpine

WORKDIR /usr/src/app

# install application dependencies
COPY package*.json ./
RUN npm install

# copy app files
COPY . .

EXPOSE 3000

CMD ["node", "server.js"]