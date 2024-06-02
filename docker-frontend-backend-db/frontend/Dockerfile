FROM node:14-alpine

WORKDIR /app

# add '/app/node_modules/.bin' to $PATH
ENV PATH /app/node_modules/.bin:$PATH

# install application dependencies
COPY package*.json ./
RUN npm install

# copy app files
COPY . .

EXPOSE 3000
CMD ["npm", "start"]