FROM node:latest

WORKDIR /app

ENV NODE_ENV=production

COPY package*json ./

RUN npm ci --only=production

EXPOSE 3000

USER node

COPY --chown=node:node ./src .

CMD [ "node" , "index.js" ]





