FROM jenkins/jenkins:lts-jdk25
USER root
# Corporate TLS inspection (Netskope): trust its CA in the JVM and OS stores
COPY certs/*.crt /usr/local/share/ca-certificates/
RUN update-ca-certificates \
 && for c in /usr/local/share/ca-certificates/*.crt; do \
      keytool -importcert -noprompt -trustcacerts -cacerts -storepass changeit -file "$c" -alias "$(basename $c)"; \
    done
USER jenkins
