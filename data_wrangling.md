##Data Wrangling

Below, we wrangle the road and crash data into a PostGIS database so we can count crashes near road segments and map them. Start by creating a new Postgres Database with the template_gis template. 

Historical road data is available from http://www.pasda.psu.edu/data/padot/state/historic/PaStateRoads/. After comparing this to the traffic data, they seem to have very similar coverage on roads. 

To figure out the projection, paste the contents of the .prj into http://prj2epsg.org/search. Looks like it's 4269, NAD83_GEO. Then, import the data into the traffic table of your PostGIS database. 

Bash
```bash
/Library/PostgreSQL/9.4/bin/shp2pgsql -I -s 4269 ~/Downloads/PaStateRoads2014_02/PaStateRoads2014_02.shp public.roads_2014 | psql -p 5432 -d crashes -U crashes

/Library/PostgreSQL/9.4/bin/shp2pgsql -I -s 4269 ~/Downloads/PaStateRoads2013_02/PaStateRoads2013_02.shp public.roads_2013 | psql -p 5432 -d crashes -U crashes

/Library/PostgreSQL/9.4/bin/shp2pgsql -I -s 4269 ~/Downloads/PaStateRoads2012_01/PaStateRoads2012_01.shp public.roads_2012 | psql -p 5432 -d crashes -U crashes

/Library/PostgreSQL/9.4/bin/shp2pgsql -I -s 4269 ~/Downloads/PaStateRoads2011_01/PaStateRoads2011_01.shp public.roads_2011 | psql -p 5432 -d crashes -U crashes

/Library/PostgreSQL/9.4/bin/shp2pgsql -I -s 4269 ~/Downloads/PaStateRoads2010_01/PaStateRoads2010_10.shp public.roads_2010 | psql -p 5432 -d crashes -U crashes
```

Lets drop rows that aren't in Allegheny County to reduce the database size.

```sql
DELETE FROM roads_2014 WHERE cty_code != '02';
DELETE FROM roads_2013 WHERE cty_code != '02';
DELETE FROM roads_2012 WHERE cty_code != '02';
DELETE FROM roads_2011 WHERE cty_code != '02';
DELETE FROM roads_2010 WHERE cty_code != '02';

ALTER TABLE roads_2014 ADD COLUMN year integer;
UPDATE roads_2014 SET year = 2014;

ALTER TABLE roads_2013 ADD COLUMN year integer;
UPDATE roads_2013 SET year = 2013;

ALTER TABLE roads_2012 ADD COLUMN year integer;
UPDATE roads_2012 SET year = 2012;

ALTER TABLE roads_2011 ADD COLUMN year integer;
UPDATE roads_2011 SET year = 2011;

ALTER TABLE roads_2010 ADD COLUMN year integer;
UPDATE roads_2010 SET year = 2010;
```

Create a new table for all the road data. 

```sql
DROP TABLE IF EXISTS roads;
CREATE TABLE roads AS SELECT * FROM roads_2014 WHERE 1=2;
ALTER TABLE roads ADD COLUMN year integer;
ALTER TABLE roads OWNER TO crashes;

INSERT INTO roads select * from roads_2014; 
INSERT INTO roads select * from roads_2013; 
INSERT INTO roads select * from roads_2012; 
INSERT INTO roads select * from roads_2011; 
INSERT INTO roads select * from roads_2010; 

CREATE INDEX roads_geom_idx
  ON roads
  USING gist
  (geom);

```


We have AADT for several years. Let's average them together. There's one complication. The road segments aren't exactly the same. So first, we'll average the traffic volumes by State Route Number, Segment Number, and Side Indicator. Then we'll merge in the most recent years of geometry from the roads table.

```sql
DROP TABLE IF EXISTS road_stats;

CREATE TABLE road_stats 
AS (SELECT
        st_rt_no, 
        seg_no,
        side_ind,
        AVG(cur_aadt) as cur_aadt_avg
FROM
        roads
WHERE 
        cur_aadt > 0
GROUP BY
        st_rt_no, 
        seg_no,
        side_ind
);
```

Let's add the latest geometry and other road characteristics.

```sql
ALTER TABLE road_stats ADD COLUMN street_nam varchar(25);
ALTER TABLE road_stats ADD COLUMN lane_cnt numeric;
ALTER TABLE road_stats ADD COLUMN total_widt numeric;
ALTER TABLE road_stats ADD COLUMN seg_lngth_ numeric;
ALTER TABLE road_stats ADD COLUMN geom geometry(MULTILINESTRING, 4269);

UPDATE road_stats s
    SET
        street_nam = u.street_nam,
        lane_cnt = u.lane_cnt, 
        total_widt = u.total_widt,
        seg_lngth_ = u.seg_lngth_,
        geom = u.geom
FROM
(
    SELECT
        r.geom, r.gid, r.st_rt_no, r.seg_no, r.side_ind, r.street_nam, r.lane_cnt, r.total_widt, r.seg_lngth_
    FROM 
    (
    SELECT
            MAX(year) as year,
            st_rt_no, 
            seg_no,
            side_ind
        FROM
            roads
        GROUP BY
            st_rt_no, 
            seg_no,
            side_ind
    ) as m,
     roads r
     WHERE 
        m.year = r.year AND
        m.st_rt_no = r.st_rt_no AND
        m.seg_no = r.seg_no AND
        m.side_ind = r.side_ind
) as u
WHERE
    s.st_rt_no = u.st_rt_no AND 
    s.seg_no = u.seg_no AND
    s.side_ind = u.side_ind;
```

Let's add an id (primary key) to the road_stats table that we can reference from other tables. Let's also transform the geometries to so we can specify the distance to them in feet.

```sql
ALTER TABLE road_stats ADD COLUMN id SERIAL PRIMARY KEY;

ALTER TABLE road_stats ADD COLUMN geom_feet geometry(MULTILINESTRING,2272);
UPDATE road_stats SET geom_feet = ST_Transform(geom, 2272);

DROP INDEX IF EXISTS road_stats_geom_feet_idx;
CREATE INDEX roads_geom_feet_idx
  ON road_stats
  USING gist
  (geom_feet);
```

Create a crashes table in our database.

SQL
```sql
DROP TABLE IF EXISTS crashes_2014;
DROP TABLE IF EXISTS crashes_2013;
DROP TABLE IF EXISTS crashes_2012;
DROP TABLE IF EXISTS crashes_2011;
DROP TABLE IF EXISTS crashes_2010;

CREATE TABLE crashes_2014 (
    crash_crn integer,
    district char(2),
    crash_county  char(2),
    municipality varchar(5),
    police_agcy varchar(5),
    crash_year integer,
    crash_month integer,
    day_of_week integer,
    time_of_day integer,
    hour_of_day integer,
    illumination integer,
    weather integer,
    road_condition integer,
    collision_type integer,
    relation_to_road integer,
    intersect_type integer,
    tcd_type integer,
    urban_rural integer,
    location_type integer,
    sch_bus_ind char(1),
    sch_zone_ind char(1),
    total_units integer,
    person_count integer,
    vehicle_count integer,
    automobile_count integer,
    motorcycle_count integer,
    bus_count integer,
    small_truck_count integer,
    heavy_truck_count integer,
    suv_count integer,
    van_count integer,
    bicycle_count integer,
    fatal_count integer,
    injury_count integer,
    maj_inj_count integer,
    mod_inj_count integer,
    min_inj_count integer,
    unk_inj_deg_count integer,
    unk_inj_per_count integer,
    unbelted_occ_count integer,
    unb_death_count integer,
    unb_maj_inj_count integer,
    belted_death_count integer,
    belted_maj_inj_count integer,
    mcycle_death_count integer,
    mcycle_maj_inj_count integer,
    bicycle_death_count integer,
    bicycle_maj_inj_count integer,
    ped_count integer,
    ped_death_count integer,
    ped_maj_inj_count integer,
    comm_veh_count integer,
    max_severity_level integer,
    driver_count_16yr integer,
    driver_count_17yr integer,
    driver_count_18yr integer,
    driver_count_19yr integer,
    driver_count_20yr integer,
    driver_count_50_64yr integer,
    driver_count_65_74yr integer,
    driver_count_75plus integer,
    latitude varchar(20),
    longitude varchar(20),
    dec_lat numeric,
    dec_long numeric,
    est_hrs_closed integer,
    lane_closed integer,
    ln_close_dir integer,
    ntfy_hiwy_maint char(1),
    rdwy_surf_type_cd integer,
    spec_juris_cd integer,
    tcd_func_cd integer,
    tfc_detour_ind char(1),
    work_zone_ind char(1),
    work_zone_type integer,
    work_zone_loc integer,
    cons_zone_spd_lim integer,
    workers_pres char(1),
    wz_close_detour char(1),
    wz_flagger char(1),
    wz_law_offcr_ind char(1),
    wz_ln_closure char(1),
    wz_moving char(1),
    wz_other char(1),
    wz_shlder_mdn char(1),
    flag_crn integer,
    interstate integer,
    state_road integer,
    local_road integer,
    local_road_only integer,
    turnpike integer,
    wet_road integer,
    snow_slush_road integer,
    icy_road integer,
    sudden_deer integer,
    shldr_related integer,
    rear_end integer,
    ho_oppdir_sdswp integer,
    hit_fixed_object integer,
    sv_run_off_rd integer,
    work_zone integer,
    property_damage_only integer,
    fatal_or_maj_inj integer,
    injury integer,
    fatal integer,
    non_intersection integer,
    intersection integer,
    signalized_int integer,
    stop_controlled_int integer,
    unsignalized_int integer,
    school_bus integer,
    school_zone integer,
    hit_deer integer,
    hit_tree_shrub integer,
    hit_embankment integer,
    hit_pole integer,
    hit_gdrail integer,
    hit_gdrail_end integer,
    hit_barrier integer,
    hit_bridge integer,
    overturned integer,
    motorcycle integer,
    bicycle integer,
    hvy_truck_related integer,
    vehicle_failure integer,
    train_trolley integer,
    phantom_vehicle integer,
    alcohol_related integer,
    drinking_driver integer,
    underage_drnk_drv integer,
    unlicensed integer,
    cell_phone integer,
    no_clearance integer,
    running_red_lt integer,
    tailgating integer,
    cross_median integer,
    curve_dvr_error integer,
    limit_65mph integer,
    speeding integer,
    speeding_related integer,
    aggressive_driving integer,
    fatigue_asleep integer,
    driver_16yr integer,
    driver_17yr integer,
    driver_65_74yr integer,
    driver_75plus integer,
    unbelted integer,
    pedestrian integer,
    distracted integer,
    curved_road integer,
    driver_18yr integer,
    driver_19yr integer,
    driver_20yr integer,
    driver_50_64yr integer,
    vehicle_towed integer,
    fire_in_vehicle integer,
    hit_parked_vehicle integer,
    mc_drinking_driver integer,
    drugged_driver integer,
    injury_or_fatal integer,
    comm_vehicle integer,
    impaired_driver integer,
    deer_related integer,
    drug_related integer,
    hazardous_truck integer,
    illegal_drug_related integer,
    illumination_dark integer,
    minor_injury integer,
    moderate_injury integer,
    major_injury integer,
    nhtsa_agg_driving integer,
    psp_reported integer,
    running_stop_sign integer,
    train integer,
    trolley integer,
    roadway_crn integer,
    rdwy_seq_num integer,
    adj_rdwy_seq integer,
    access_ctrl integer,
    roadway_county integer,
    lane_count integer,
    rdwy_orient char(1),
    road_owner integer,
    route text,
    speed_limit integer,
    segment integer,
    offset_number integer,
    street_name text
);

CREATE TABLE crashes_2013 AS SELECT * FROM crashes_2014 WHERE 1=2;
CREATE TABLE crashes_2012 AS SELECT * FROM crashes_2014 WHERE 1=2;
CREATE TABLE crashes_2011 AS SELECT * FROM crashes_2014 WHERE 1=2;
CREATE TABLE crashes_2010 AS SELECT * FROM crashes_2014 WHERE 1=2;

COPY crashes_2014 FROM '/Users/Shared/2014alcocrash.csv' WITH DELIMITER ',' NULL '' CSV HEADER;
COPY crashes_2013 FROM '/Users/Shared/2013alcocrash.csv' WITH DELIMITER ',' NULL '' CSV HEADER;
COPY crashes_2012 FROM '/Users/Shared/2012alcocrash.csv' WITH DELIMITER ',' NULL '' CSV HEADER;
COPY crashes_2011 FROM '/Users/Shared/2011alcocrash.csv' WITH DELIMITER ',' NULL '' CSV HEADER;
COPY crashes_2010 FROM '/Users/Shared/2010alcocrash.csv' WITH DELIMITER ',' NULL '' CSV HEADER;

ALTER TABLE crashes_2014 ADD COLUMN year integer;
UPDATE crashes_2014 SET year = 2014;

ALTER TABLE crashes_2013 ADD COLUMN year integer;
UPDATE crashes_2013 SET year = 2013;

ALTER TABLE crashes_2012 ADD COLUMN year integer;
UPDATE crashes_2012 SET year = 2012;

ALTER TABLE crashes_2011 ADD COLUMN year integer;
UPDATE crashes_2011 SET year = 2011;

ALTER TABLE crashes_2010 ADD COLUMN year integer;
UPDATE crashes_2010 SET year = 2010;
```

Let's put together all the crash data into the same table

```sql
DROP TABLE IF EXISTS crashes;
CREATE TABLE crashes AS SELECT * FROM crashes_2014 WHERE 1=2;
ALTER TABLE crashes OWNER TO crashes;

INSERT INTO crashes select * from crashes_2014; 
INSERT INTO crashes select * from crashes_2013; 
INSERT INTO crashes select * from crashes_2012; 
INSERT INTO crashes select * from crashes_2011; 
INSERT INTO crashes select * from crashes_2010; 
```

Let's add crash point geometry to the database.

```sql
ALTER TABLE crashes ADD COLUMN geom geometry(POINT,4326);
UPDATE crashes SET geom = ST_SetSRID( ST_MakePoint(dec_long, dec_lat), 4326);

DROP INDEX IF EXISTS crashes_geom_idx;
CREATE INDEX crashes_geom_idx
  ON crashes
  USING gist
  (geom);

ALTER TABLE crashes ADD COLUMN geom_feet geometry(POINT,2272);
UPDATE crashes SET geom_feet = ST_Transform(geom, 2272);

DROP INDEX IF EXISTS crashes_geom_feet_idx;
CREATE INDEX crashes_geom_feet_idx
  ON crashes
  USING gist
  (geom_feet);
```

Next, we want to update every crash with the id of the closest road_stats segment for the given year. We don't want to compute the distances from each of the 12,000 crash points to each of the 2700 road segments. So first, we find all roads within 50 feet of the crash, compute the distances, and take the minimum.

SQL
```sql
ALTER TABLE crashes ADD COLUMN road_stats_id integer ;
ALTER TABLE crashes ADD COLUMN road_stats_id_distance numeric;
UPDATE crashes 
    SET road_stats_id = d.id, road_stats_id_distance = d.distance 
    FROM (
       SELECT DISTINCT ON (crash_crn)
            crash_crn,
            id,
            MIN(ST_Distance(crashes_geom, road_stats_geom)) as distance
        FROM
        (
            SELECT 
                t.id,
                c.crash_crn,
                c.geom_feet as crashes_geom,
                t.geom_feet as road_stats_geom
            FROM 
                road_stats t, 
                crashes c 
            WHERE
                 ST_DWithin(t.geom_feet, c.geom_feet, 50)
        ) as n
        GROUP BY 
            crash_crn, id
        ORDER BY crash_crn, distance
    ) as d 
WHERE crashes.crash_crn = d.crash_crn;
```


Export the crashes above to a csv, now that they're associated with a segment.

```sql
COPY (
    SELECT 
        road_stats_id,
        road_stats_id_distance,
        year,
        dec_lat, 
        dec_long,
        fatal_count,
        maj_inj_count,
        bicycle_death_count,
        bicycle_maj_inj_count,
        ped_death_count,
        ped_maj_inj_count,
        time_of_day,
        weather,
        relation_to_road,
        alcohol_related,
        speeding,
        speeding_related,
        aggressive_driving

    FROM crashes 
    WHERE 
        road_crash_stats_id IS NOT NULL AND
        fatal_count > 0 OR
        maj_inj_count > 0 OR
        bicycle_death_count > 0 OR
        bicycle_maj_inj_count > 0 OR
        ped_death_count > 0 OR
        ped_maj_inj_count > 0
) TO '/Users/Shared/crashes.csv'  DELIMITER ',' CSV HEADER;
```


Now, let's aggregate data of interest and append it to the road_stats table.

SQL
```sql
-- These are indicator variables.
ALTER TABLE road_stats ADD COLUMN aggressive_driving numeric;
ALTER TABLE road_stats ADD COLUMN alcohol_related numeric;
ALTER TABLE road_stats ADD COLUMN automobile_count numeric;
ALTER TABLE road_stats ADD COLUMN bicycle numeric;
ALTER TABLE road_stats ADD COLUMN deer_related numeric;
ALTER TABLE road_stats ADD COLUMN injury numeric;
ALTER TABLE road_stats ADD COLUMN fatal numeric;
ALTER TABLE road_stats ADD COLUMN minor_injury numeric;
ALTER TABLE road_stats ADD COLUMN moderate_injury numeric;
ALTER TABLE road_stats ADD COLUMN major_injury numeric;
ALTER TABLE road_stats ADD COLUMN speeding numeric;
ALTER TABLE road_stats ADD COLUMN pedestrian numeric;
ALTER TABLE road_stats ADD COLUMN speeding_related numeric;
-- These are count variables.
ALTER TABLE road_stats ADD COLUMN fatal_count numeric;
ALTER TABLE road_stats ADD COLUMN injury_count numeric;
ALTER TABLE road_stats ADD COLUMN maj_inj_count numeric;
ALTER TABLE road_stats ADD COLUMN mod_inj_count numeric;
ALTER TABLE road_stats ADD COLUMN min_inj_count numeric;
ALTER TABLE road_stats ADD COLUMN bicycle_count numeric;
ALTER TABLE road_stats ADD COLUMN bicycle_death_count numeric;
ALTER TABLE road_stats ADD COLUMN bicycle_maj_inj_count numeric;
ALTER TABLE road_stats ADD COLUMN ped_count numeric;
ALTER TABLE road_stats ADD COLUMN ped_death_count numeric;
ALTER TABLE road_stats ADD COLUMN ped_maj_inj_count numeric;
ALTER TABLE road_stats ADD COLUMN person_count numeric;
ALTER TABLE road_stats ADD COLUMN vehicle_count numeric;

UPDATE road_stats 
    SET 
        aggressive_driving = 0,
        alcohol_related = 0,
        automobile_count = 0,
        bicycle = 0,
        deer_related = 0,
        injury = 0,
        fatal = 0,
        minor_injury = 0,
        moderate_injury = 0,
        major_injury = 0,
        speeding = 0,
        pedestrian = 0,
        speeding_related = 0,

        fatal_count = 0,
        injury_count = 0,
        maj_inj_count = 0,
        mod_inj_count = 0,
        min_inj_count = 0,
        bicycle_count = 0,
        bicycle_death_count = 0,
        bicycle_maj_inj_count = 0,
        ped_count = 0,
        ped_death_count = 0,
        ped_maj_inj_count = 0,
        person_count = 0,
        vehicle_count = 0
;

UPDATE road_stats 
    SET 
        aggressive_driving = s.aggressive_driving,
        alcohol_related = s.alcohol_related,
        automobile_count = s.automobile_count,
        bicycle = s.bicycle,
        deer_related = s.deer_related,
        injury = s.injury,
        fatal = s.fatal,
        minor_injury = s.minor_injury,
        moderate_injury = s.moderate_injury,
        major_injury = s.major_injury,
        speeding = s.speeding,
        pedestrian = s.pedestrian,
        speeding_related = s.speeding_related,

        fatal_count = s.fatal_count,
        injury_count = s.injury_count,
        maj_inj_count = s.maj_inj_count,
        mod_inj_count = s.mod_inj_count,
        min_inj_count = s.min_inj_count,
        bicycle_count = s.bicycle_count,
        bicycle_death_count = s.bicycle_death_count,
        bicycle_maj_inj_count = s.bicycle_maj_inj_count,
        ped_count = s.ped_count,
        ped_death_count = s.ped_death_count,
        ped_maj_inj_count = s.ped_maj_inj_count,
        person_count = s.person_count,
        vehicle_count = s.vehicle_count
    FROM (
        SELECT 
            road_stats_id,
            SUM(aggressive_driving) as aggressive_driving,
            SUM(alcohol_related) as alcohol_related,
            SUM(automobile_count) as automobile_count,
            SUM(bicycle) as bicycle,
            SUM(deer_related) as deer_related,
            SUM(injury) as injury,
            SUM(fatal) as fatal,
            SUM(minor_injury) as minor_injury,
            SUM(moderate_injury) as moderate_injury,
            SUM(major_injury) as major_injury,
            SUM(speeding) as speeding,
            SUM(pedestrian) as pedestrian,
            SUM(speeding_related) as speeding_related,

            SUM(fatal_count) as fatal_count,
            SUM(injury_count) as injury_count,
            SUM(maj_inj_count) as maj_inj_count,
            SUM(mod_inj_count) as mod_inj_count,
            SUM(min_inj_count) as min_inj_count,
            SUM(bicycle_count) as bicycle_count,
            SUM(bicycle_death_count) as bicycle_death_count,
            SUM(bicycle_maj_inj_count) as bicycle_maj_inj_count,
            SUM(ped_count) as ped_count,
            SUM(ped_death_count) as ped_death_count,
            SUM(ped_maj_inj_count) as ped_maj_inj_count,
            SUM(person_count) as person_count,
            SUM(vehicle_count) as vehicle_count
        FROM 
            crashes c 
        GROUP BY 
            road_stats_id
    ) as s
WHERE
    road_stats.id = s.road_stats_id;

ALTER TABLE road_stats ADD COLUMN dly_vmt_avg numeric;
UPDATE road_stats SET dly_vmt_avg = (seg_lngth_ / 5280) * cur_aadt_avg;
```


```bash
ogr2ogr -f GeoJSON road_stats.json "PG:host=localhost dbname=crashes user=crashes password=crashes" -sql 'SELECT id, st_rt_no, seg_no, cur_aadt_avg, street_nam, lane_cnt, seg_lngth_, geom, dly_vmt_avg, fatal_count, maj_inj_count, bicycle_death_count, bicycle_maj_inj_count, ped_death_count, ped_maj_inj_count FROM road_stats;'
```



Refactoring the junk below...






```sql
ALTER TABLE road_stats
    AVG( (fatal_count * 100 000 000) / (dly_vmt  * 365 ) ) as fatal_count_vmt_avg,
        AVG( (maj_inj_count * 1000000000) / (dly_vmt  * 365 ) ) as maj_inj_count_vmt_rate,
    
        AVG( (bicycle_death_count * 1000000000) / (dly_vmt  * 365 ) ) as bicycle_death_count_vmt_rate,
        AVG( (bicycle_maj_inj_count * 1000000000) / (dly_vmt  * 365 ) ) bicycle_maj_inj_count_vmt_avg,

        AVG( (ped_death_count * 1000000000) / (dly_vmt  * 365 ) ) as ped_death_count_vmt_avg,
        AVG( (ped_maj_inj_count * 1000000000) / (dly_vmt  * 365 ) ) as ped_maj_inj_count_vmt_avg,

        SUM(fatal_count ) as fatal_count_sum,
        SUM(maj_inj_count ) as maj_inj_count_sum,
    
        SUM(bicycle_death_count ) as bicycle_death_count_sum,
        SUM(bicycle_maj_inj_count ) bicycle_maj_inj_count_sum,

        SUM(ped_death_count ) as ped_death_count_sum,
        SUM(ped_maj_inj_count ) as ped_maj_inj_count_sum

```



Export the data to json to view with Leaflet.

Bash
```bash
ogr2ogr -f GeoJSON road_crash_stats.json "PG:host=localhost dbname=crashes user=crashes password=crashes" -sql 'select * from road_crash_stats;'
```

Add a sequence to the road crash stats table so we can associate it with the data from the crashes table.

```sql
CREATE SEQUENCE road_crash_stats_seq;
ALTER TABLE road_crash_stats ADD COLUMN id integer DEFAULT nextval('road_crash_stats_seq');

ALTER TABLE crashes ADD COLUMN road_crash_stats_id integer;
ALTER TABLE crashes ADD COLUMN road_crash_stats_distance numeric;

UPDATE crashes 
    SET road_crash_stats_id = d.id, road_crash_stats_distance = d.distance 
    FROM (
       SELECT DISTINCT ON (crash_crn)
            crash_crn,
            id,
            MIN(ST_Distance(crash_geom, ST_Transform(roads_geom,2272))) as distance
        FROM
        (
            SELECT 
                id,
                crash_crn,
                c.geom_feet as crash_geom,
                t.geom as roads_geom
            FROM 
                road_crash_stats t, 
                crashes c 
            WHERE
                 ST_DWithin(ST_Transform(t.geom,2272), c.geom_feet, 50)
        ) as n
        GROUP BY 
            crash_crn, id
        ORDER BY crash_crn, distance
    ) as d 
WHERE crashes.crash_crn = d.crash_crn;
```


