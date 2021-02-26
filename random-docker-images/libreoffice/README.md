# LibreOffice Dockerfiles

## Versions

Check the sub directories which index which libreOffice version will be installed inside the Docker image.

## Usage Examples

Build it

`docker build . -t sirreeall/libreoffice:1.0`

Getting inside the container

`$ docker run -it sirreeall/libreoffice:1.0 /bin/sh`

Detaching from the container without stopping 

`Ctrl-P Ctrl-Q`

Performing a basic convert

`soffice --convert-to pdf * --headless`

NOTE: the * will attempt to convert any of the files found in the current directory. You will need to make sure you copy you test documents into a running container