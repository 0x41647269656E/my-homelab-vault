# Dashboard

> Catalogue de services (Obsidian + Dataview).  
> Astuce : utilise la vue “Table” pour trier par colonnes.

## Tous les services

```dataview
table
  file.link as Service,
  category as Category,
  integration_status as Integration,
  status as Status,
  port as Port,
  external_url as Website,
  github_url as GitHub
where type = "service"
sort name asc
```

## Par catégorie

```dataview
table file.link as Service, integration_status as Integration, status as Status
where type = "service"
group by category
sort key asc
```

## À intégrer (todo)

```dataview
table file.link as Service, category
where type="service" and integration_status != "integrated"
sort category asc
```
