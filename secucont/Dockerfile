FROM alpine
RUN adduser -D appuser
USER appuser
CMD ["sh", "-c", "echo Container sécurisé! && echo Utilisateur: $(whoami) && echo UID: $(id -u)"]

