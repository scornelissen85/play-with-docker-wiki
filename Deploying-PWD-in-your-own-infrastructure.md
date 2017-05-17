


# Cleaning up everything PWD creates

```bash
# Deletes all the exited or created containers with it's associated volume
docker rm -fv $(docker ps -aq --filter status=exited --filter status=created)

# Remove all PWD networks
docker network rm $(docker network ls | grep "-" | cut -d ' ' -f1)
```