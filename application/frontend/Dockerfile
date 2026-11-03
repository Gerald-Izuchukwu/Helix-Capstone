FROM nginx:alpine

COPY index.html /usr/share/nginx/html

RUN addgroup --system appgroup && \
    adduser --system --ingroup appgroup appuser && \
    chown -R appuser:appgroup /app

USER appuser

EXPOSE 80