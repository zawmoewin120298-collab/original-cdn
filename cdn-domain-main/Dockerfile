# syntax=docker/dockerfile:1

# =============================================
# Stage 1: CrimsonCF Build
# =============================================
FROM node:22-alpine AS crimsoncf-build
WORKDIR /crimsoncf

# Yarn 4.5.0 အတွက် corepack ကို အတိအကျသတ်မှတ်
RUN corepack enable && corepack prepare yarn@4.5.0 --activate

# Lock file အပါအဝင် အကုန်ကူး
COPY package.json yarn.lock .yarnrc.yml ./
COPY .yarn ./.yarn

# Dependencies install (lock file အတိုင်း)
RUN yarn config set npmRegistryServer https://registry.npmjs.org \
    && yarn install --immutable --inline-builds --network-timeout 600000

# Source ကူးပြီး build
COPY . .
RUN yarn build

# =============================================
# Stage 2: Final Image (OpenResty + Xray + CrimsonCF)
# =============================================
FROM igorbarinov/openresty-nginx-module-vts

# =============================================
# 1. Network Optimization (BBR + TCP Tuning)
# =============================================
RUN echo "net.core.somaxconn = 1024" >> /etc/sysctl.conf && \
    echo "net.core.netdev_max_backlog = 5000" >> /etc/sysctl.conf && \
    echo "net.ipv4.tcp_max_syn_backlog = 1024" >> /etc/sysctl.conf && \
    echo "net.ipv4.tcp_syncookies = 1" >> /etc/sysctl.conf && \
    echo "net.ipv4.tcp_tw_reuse = 1" >> /etc/sysctl.conf && \
    echo "net.ipv4.tcp_fin_timeout = 30" >> /etc/sysctl.conf && \
    echo "net.ipv4.tcp_keepalive_time = 300" >> /etc/sysctl.conf && \
    echo "net.ipv4.tcp_keepalive_intvl = 60" >> /etc/sysctl.conf && \
    echo "net.ipv4.tcp_keepalive_probes = 5" >> /etc/sysctl.conf && \
    echo "net.core.default_qdisc = fq" >> /etc/sysctl.conf && \
    echo "net.ipv4.tcp_congestion_control = bbr" >> /etc/sysctl.conf

# =============================================
# 2. Install Node.js + Xray
# =============================================
RUN apk --no-cache add curl unzip nodejs npm \
    && curl -L https://github.com/XTLS/Xray-core/releases/latest/download/Xray-linux-64.zip -o xray.zip \
    && unzip xray.zip -d /usr/local/bin/ \
    && chmod +x /usr/local/bin/xray \
    && rm xray.zip \
    && apk del curl unzip

# =============================================
# 3. Create directories
# =============================================
RUN mkdir -p /etc/xray /cache /usr/local/openresty/nginx/html /app/crimsoncf

# =============================================
# 4. Copy Xray & Nginx configs
# =============================================
COPY ./config.json /etc/xray/config.json
COPY ./nginx.conf /usr/local/openresty/nginx/conf/nginx.conf
COPY ./nginx_edge.conf /usr/local/openresty/nginx/conf/nginx_edge.conf
COPY ./generic_conf/ /usr/local/openresty/nginx/conf/generic_conf/
COPY ./src/ /usr/local/openresty/nginx/src/

# =============================================
# 5. Copy Website HTML
# =============================================
COPY ./my-website/ /usr/local/openresty/nginx/html/

# =============================================
# 6. Copy CrimsonCF from build stage
# =============================================
COPY --from=crimsoncf-build /crimsoncf/dist /app/crimsoncf/dist
COPY --from=crimsoncf-build /crimsoncf/server /app/crimsoncf/server
COPY --from=crimsoncf-build /crimsoncf/package.json /app/crimsoncf/package.json
COPY --from=crimsoncf-build /crimsoncf/yarn.lock /app/crimsoncf/yarn.lock

# =============================================
# 7. Install CrimsonCF dependencies (production only)
# =============================================
WORKDIR /app/crimsoncf
RUN corepack enable \
    && yarn config set npmRegistryServer https://registry.npmjs.org \
    && yarn install --production --immutable --network-timeout 600000

# =============================================
# 8. Environment Variables
# =============================================
ENV NODE_ENV=production
ENV PORT=8080
ENV SERVE_STATIC=1

# =============================================
# 9. Health Check (အသစ်ထည့်ထား)
# =============================================
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# =============================================
# 10. Permissions
# =============================================
RUN chmod 755 /cache

# =============================================
# 11. Expose ports
# =============================================
EXPOSE 443 8080 80 3128 53/udp 10001

# =============================================
# 12. Start all services
# =============================================
WORKDIR /app/crimsoncf
CMD /usr/local/bin/xray -config /etc/xray/config.json & \
    node server/index.mjs & \
    /usr/local/openresty/bin/openresty -g "daemon off;"
