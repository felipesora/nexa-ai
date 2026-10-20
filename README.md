```
git clone --recurse-submodules https://github.com/felipesora/nexa-ai.git
```

```
# develop
docker compose --env-file .env.develop up -d

# prod
docker compose --env-file .env.prod up -d
```