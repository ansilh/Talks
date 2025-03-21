# Demo 1

```bash
mkdir buildkit-demo
cd buildkit-demo
```

```bash
cat >>Dockerfile <<EOF
FROM busybox as st0
RUN echo BEGIN:st0
RUN touch /ST0 && sleep 5
RUN echo END:st0

FROM busybox as st1
RUN echo BEGIN:st1
RUN touch /ST1 && sleep 5
RUN echo END:st1

FROM busybox as st2
RUN echo BEGIN:st2
COPY --from=st0 /ST0 /ST2.A
COPY --from=st1 /ST1 /ST2.B
RUN echo END:st2
EOF
```

```bash
docker builder prune -a
```
### Legacy build
```bash
export DOCKER_BUILDKIT=0
```

```bash
time docker build .
```

### BuildKit

```bash
docker builder prune -a
```

```bash
time docker build . --progress=plain
```
