# Dockerfile comment
FROM openjdk:18.0.2.1-jdk
ARG JAR_DEST
ENV JAR_DEST ${JAR_DEST}
ARG JAR_SOURCE
ENV JAR_SOURCE ${JAR_SOURCE}
RUN mkdir -p /opt/i27/
COPY ["${JAR_SOURCE}", "/opt/i27/i27cart-product.jar"]
RUN chmod 777 /opt/i27/
WORKDIR /opt/i27
EXPOSE 8080
USER root
#CMD java -jar $JAR_DEST 
#CMD ["java", "-jar", "/opt/learner/$JAR_DEST"]
CMD ["java", "-jar", "/opt/i27/i27cart-product.jar"]