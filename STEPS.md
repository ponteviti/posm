# Manual Steps

For the purpose of these steps, we're provisioning data for a past project in
Dvizarasekwa, Zimbabwe. The _Deployment AOI_ is `[30.904111862182614,
-17.821343531895728, 30.95578193664551, -17.778602961844793]`.

## Initial Data Gathering

### Generate MBTiles

```bash
nvm use 5
npm install tl tilelive-http mbtiles
node_modules/.bin/tl copy \
  http://a.tile.openstreetmap.org/{z}/{x}/{y}.png \
  mbtiles://./osm_dvizarasekwa_z14-20.mbtiles \
  -b "30.904111862182614 -17.821343531895728 30.95578193664551 -17.778602961844793" \
  -z 14 \
  -Z 20
```

### Fetch OSM XML

```bash
curl -XPOST \
  --compressed \
  -d "data=(node(-17.821343531895728, 30.904111862182614, -17.778602961844793, 30.95578193664551); <; >>; >;); out meta qt;" \
  http://overpass-api.de/api/interpreter > dvizarasekwa.osm
```

### Convert OSM XML → PBF

```bash
osmosis --read-xml dvizarasekwa.osm --wb dvizarasekwa.pbf
```

## Initial Data Deployment

```bash
sudo -s
mkdir -p /opt/data/tiles
mkdir -p /opt/data/osm
mv /path/to/dvizarasekwa_z14-20.mbtiles /opt/data/tiles
mv /path/to/dvizarasekwa.pbf /opt/data/osm
```

### Populate APIDB

**TBD** (requires macrocosm)

### Populate Rendering DB

```bash
mem=$(awk 'NR == 1 { print int($2*.9/1024) } ' /proc/meminfo)
sudo -u gis \
  osm2pgsql \
    --create \
    --hstore-all \
    --hstore-add-index \
    --extra-attributes \
    --slim \
    --drop \
    --unlogged \
    --database='gis' \
    -C $mem \
    --number-processes $(nproc) /opt/data/osm/dvizarasekwa.pbf
```

### Restart Tile Server

```bash
sudo -s
cat <<EOF > /etc/tessera.conf.json
{
  "/tl": "mapnik://./project.xml",
  "/osm": "mbtiles:///opt/data/tiles/osm_dvizarasekwa_z14-20.mbtiles"
}
EOF
service tessera restart
```

Add this to `/etc/nginx/sites-enabled/posm` to proxy OSM tiles:

```
location /osm {
  proxy_pass http://127.0.0.1:8082;
}
```

```bash
sudo service nginx restart
```

OSM tiles will now be available at [`posm.local/osm/`](http://posm.local/osm/)
(hostname dependent on your configuration). It will auto-center to the center of
the deployment AOI.

Locally-rendered tiles will now be available at
[`posm.local/tl/#16/-17.7990/30.9290`](http://posm.local/tl/#16/-17.799/30.929)
(again, hostname dependent on your configuration). The location in the URL hash
is required until the map style is correctly configured with a center.