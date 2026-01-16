FROM node:18-alpine

WORKDIR /app

# Copy only package files first (better caching)
COPY app/package*.json ./

RUN npm install --omit=dev

# Copy application source
COPY app/ .

EXPOSE 3000

CMD ["node", "app.js"]
