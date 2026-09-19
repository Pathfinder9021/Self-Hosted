# Docker Templates

This is a respository with predefined templates for running service in Docker.

ttps://raw.githubusercontent.com/Pathfinder9021/Self-Hosted/refs/heads/main/Docker/Templates/templates.json

## Prerequisites

- Temples expects a Docker network named "public-apps" to exist.

## How to add new template

1. Create a new folder with with a Docker compose file.
2. Add template to `templates.json` file.
```
    {
      "type": 3,
      "title": "<Title>",
      "description": "<Desription>",
      "logo": "<Logo>,
      "categories": ["<Category>"],
      "repository": {
        "url": "https://github.com/Pathfinder9021/Self-Hosted/Docker./Templates",
        "stackfile": "<Folder>/compose.yaml"
      }
    },
```
__Note__: For more information read [Portainer documentation](https://docs.portainer.io/user/docker/templates/custom).

__Note__: An icon for an application can be found on [dashboardicons.com](https://dashboardicons.com/).
