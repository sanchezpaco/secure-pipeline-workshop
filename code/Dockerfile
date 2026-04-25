FROM node:20-alpine

# Bump bundled npm so its tinyglobby > picomatch chain ships >= 4.0.4
# (fixes GHSA-c2c7-rcm5-vvqj high finding picked up by container scanners).
RUN npm install -g npm@11.13.0 && \
  npm cache clean --force

RUN addgroup -g 1001 -S nodejs && \
  adduser -S nextjs -u 1001 -G nodejs

WORKDIR /app

COPY package*.json ./

RUN npm ci --only=production && \
  npm cache clean --force

COPY --chown=nextjs:nodejs src/ ./src/

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (res) => { process.exit(res.statusCode === 200 ? 0 : 1) })"

USER nextjs

CMD ["npm", "start"]