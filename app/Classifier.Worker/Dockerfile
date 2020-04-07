# 3.1.200-alpine3.11 or 3.1.200-buster 
FROM mcr.microsoft.com/dotnet/core/sdk:3.1.200-alpine3.11 AS build-env
WORKDIR /app
# copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore -r linux-x64
# copy everything else and build
COPY ./ClassificationResult.cs ./
COPY ./ImageMetadata.cs ./
COPY ./ImageUtil.cs ./
COPY ./Program.cs ./
COPY ./Startup.cs ./
COPY ./WorkerHostedService.cs ./
RUN dotnet publish -c Release -r linux-x64 -o out --self-contained true /p:PublishTrimmed=true

# build runtime image : 3.1.2-alpine3.11 or 3.1.2-buster-slim
FROM mcr.microsoft.com/dotnet/core/runtime-deps:3.0.1-buster-slim
# Add non-root user
RUN groupadd -g 99 appuser && useradd -r -u 99 -g appuser appuser
WORKDIR /app
ENV ASPNETCORE_URLS=http://+:5000
EXPOSE 5000
COPY --chown=appuser:appuser ./assets ./assets
COPY --chown=appuser:appuser --from=build-env /app/out ./
#RUN chown -R appuser:appuser /app
USER appuser
ENTRYPOINT ["./Classifier.Worker"]
