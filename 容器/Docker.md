
# Docker内容还原方法

## undocker

```bash
pip install git+https://github.com/larsks/undocker/ 
docker save IMAGE_NAME | undocker -i -o IMAGE_NAME
```

## dockerfile-from-image

```bash
https://github.com/CenturyLinkLabs/dockerfile-from-image
docker run -v /var/run/docker.sock:/var/run/docker.sock centurylink/dockerfile-from-image <IMAGE_TAG_OR_ID>
```

## shell

dockerfile2.sh

```bash
#!/bin/bash
#########################################################################
# File Name: dockerfile.sh
# Author: www.linuxea.com
# Version: 1
# Created Time: Thu 14 Feb 2019 10:52:01 AM CST
#########################################################################
case "$OSTYPE" in
 linux*)
 docker history --no-trunc --format "{{.CreatedBy}}" $1 | # extract information from layers
 tac | # reverse the file
 sed 's,^\(|3.*\)\?/bin/\(ba\)\?sh -c,RUN,' | # change /bin/(ba)?sh calls to RUN
 sed 's,^RUN #(nop) *,,' | # remove RUN #(nop) calls for ENV,LABEL...
 sed 's, *&& *, \\\n \&\& ,g' # pretty print multi command lines following Docker best practices
 ;;
 darwin*)
 docker history --no-trunc --format "{{.CreatedBy}}" $1 | # extract information from layers
 tail -r | # reverse the file
 sed -E 's,^(\|3.*)?/bin/(ba)?sh -c,RUN,' | # change /bin/(ba)?sh calls to RUN
 sed 's,^RUN #(nop) *,,' | # remove RUN #(nop) calls for ENV,LABEL...
 sed $'s, *&& *, \\\ \\\n \&\& ,g' # pretty print multi command lines following Docker best practices
 ;;
 *)
 echo "unknown OSTYPE: $OSTYPE"
 ;;
esac
```

```bash
bash dockerfile2.sh marksugar/redis:5.0.0
```

## docker history

```bash
docker history <image>
```