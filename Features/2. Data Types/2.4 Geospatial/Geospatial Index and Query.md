## Geospatial Index and Query

使用 Aerospike 地理空间存储(geospatial storage)，索引(indexing)和查询(query)可以对区域内的点、包含点的区域和半径内的点进行快速查询。

##### Contents

- [Underlying technologies](#underlying-technologies)
- [Use Cases](#use-cases)
- [Geospatial Data](#geospatial-data)
- [Geospatial Index](#geospatial-index)
- [Geospatial Query](#geospatial-query)
  - [Points-within-Region — Circle Query](#points-within-region-circle-query)
  - [Region-Contains-Points Query](#region-contains-points-query)
- [Query Filters](#query-filters)
- [Index on list/map](#index-on-list-map)
- [Aerospike GeoJSON Extension](#aerospike-geojson-extension)
- [GeoJSON Parsing](#geojson-parsing)
- [Create a Geospatial Application](#create-a-geospatial-application)
- [Known Limitations](#known-limitations)

---

### <span id="underlying-technologies"> Underlying technologies </span>

Aerospike 地理空间功能依赖于一下技术：

- **GeoJSON format** 来指定和存储 [GeoJSON geometry objects](http://geojson.org/geojson-spec.html#geometry-objects) ，并实现数据标准化和互操作性。请参阅 [GeoJSON Format Specification](http://geojson.org/geojson-spec.html) 和 [GeoJSON IETF WG](https://github.com/geojson/draft-geojson) 。
- Aerospike [AeroCircle](https://docs.aerospike.com/docs/#aerospike-geojson-extension) 数据类型扩展了 GeoJSON 格式以存储圆和多边形。
- **S2 Spherical Geometry Library**以单维 64 位 CellID 表示映射点和区域。
- Aerospike [secondary indexes](https://docs.aerospike.com/docs/architecture/secondary-index.html) and [queries](https://docs.aerospike.com/docs/guide/query.html) 用于提高 index inserts/updates and queries 的性能和规模。

[S2 Geometry Library](https://code.google.com/archive/p/s2-geometry-library/) 是一个球面几何库，对于操纵球体（通常在地球上）上的区域和索引地理数据非常有用。

此外，请查看此 [S2 blog](http://blog.christianperone.com/2015/08/googles-s2-geometry-on-the-sphere-cells-and-hilbert-curve/) ，了解有关 S2，Hilbert Curve，和 Geospatial mapping 的详细信息。

---

#### <span id="use-cases"> User Cases </span>

- 需要高吞吐量更新车辆位置并频繁查询区域内车辆的车辆跟踪系统。
- 零售地点，可在购物者 500m 范围内找到特定的便利设施。
- 以位置为目标的出价交易，通过有效的广告活动发现该位置内的人员或设备。

请参阅这些 [Aerospike examples](https://github.com/aerospike/geospatial-samples) 。

---

#### <span id="geospatial-data"> Geospatial Data </span>

Aerospike 支持 GeoJSON 地理空间数据类型(geospatial data type)。所有地理空间功能(indexing and querying)仅在 GeoJSON data types 上执行。

GeoJSON 数据会导致对数据读取进行这种额外的处理：
- 解析 GeoJSON 文本的有效性和支持(请参阅 [GeoJSON Parsing](https://docs.aerospike.com/docs/#geojson-parsing)) 。 
- GeoJSON 文本被转换为 S2 CellID 覆盖。
- Aerospike 将覆盖的 CellID 和原始 GeoJSON 保存在数据库中。
- 应用程序只能通过客户端 API 和 UDF 子系统访问 GeoJSON 数据。

---

### <span id="geospatial-index"> Geospatial Index </span>

除了 integers and strings，Aerospike 还支持用于索引的 [Geo2DSphere](http://api.mongodb.org/csharp/current/html/M_MongoDB_Driver_IndexKeysDefinitionBuilder_1_Geo2DSphere_1.htm) 数据类型。

此示例创建 aql 脚本 Geo2DSphere 索引数据类型；
```
aql> CREATE INDEX my_geo_index ON test.demoset (geobin) GEO2DSPHERE
```

Geo2DSphere 索引的行为与其他索引类型一样：
- 扫描现有记录以检查索引 bin (上例中的 geobin) 以构建内存地理空间索引。
- 为每个节点上的数据创建一个独立的索引。
- 为后续数据插入和更新，更新索引。
- 当节点重新启动时重建索引。

---

### <span id="geospatial-query"> Geospatial Query </span>

Aerospike 支持两种地理空间查询：
- Points with a region(including circle)
- Region contains point

#### <span id="points-within-region-circle-query"> Points-within-Region — Circle Query

这个 Python 脚本示例是一个 points-within-a-region 查询。
```
def query_pwr(args,client):

    """Construct a GeoJSON region."""
    region = aerospike.GeoJSON({
        'type': 'Polygon', 
        'coordinates': [[[-122.500000,37.000000],
                         [-121.000000, 37.000000], 
                         [-121.000000, 38.080000],
                         [-122.500000, 38.080000], 
                         [-122.500000, 37.000000]]]})

    """Construct the query predicate."""
    query = client.query(args.nspace, args.set)
    predicate = aerospike.predicates.geo_within_geojson_region(LOCBIN, region.dumps())
    query.where(predicate)

    """Define callback to process query result."""
    def callback((key, metadata, record)):
        records.append(record)

    """Make the actual query!"""

    query.foreach(callback)
```

使用 AQL 的 points-within-a-region 查询示例。
- 将 point 数据插入到 aerospike。
```
aql> INSERT INTO test.testset (PK, geo_query_bin) VALUES (2, GEOJSON('{"type": "Point", "coordinates": [1,1]}'))
```
- 查询区域以查看它是否包含该点。
```
aql> SELECT * FROM test.testset WHERE geo_query_bin CONTAINS GeoJSON('{"type":"Polygon", "coordinates": [[[0,0], [0, 10], [10, 10], [10, 0], [0,0]]]}'))
+----------------------------------------------------+
| geo_query_bin                                      |
+----------------------------------------------------+
| GeoJSON('{"type": "Point", "coordinates": [1,1]}') |
+----------------------------------------------------+
1 row in set (0.004 secs)
```

#### <span id="region-contains-points-query"> Region-Contains-Points Query </span>

此示例 C++ 脚本是一个 region-contains-points 查询。
```
// Callback function to process each record response
bool
query_cb(const as_val * valp, void * udata)
{
    if (!valp)
        return true;    // query complete

    char const * valstr = NULL;

    as_record * recp = as_record_fromval(valp);
    if (!recp)
        fatal("query callback returned non-as_record object");
    valstr = as_record_get_str(recp, g_valbin);

    __sync_fetch_and_add(&g_numrecs, 1);

    cout << valstr << endl;

    return true;
}

// Main query function
void
query_prcp(aerospike * asp, double lat, double lng)
{
    char point[1024];

    // Construct a GeoJSON point.
    snprintf(point, sizeof(point),
             "{ \"type\": \"Point\", \"coordinates\": [%0.8f, %0.8f] }",
             lng, lat);

    // Construct the query object.
    as_query query;
    as_query_init(&query, g_namespace.c_str(), g_set.c_str());

    as_query_where_inita(&query, 1);
    as_query_where(&query, g_rgnbin, as_geo_contains(point));

    // Make the actual query.
    as_error err;
    if (aerospike_query_foreach(asp, &err, NULL,
                                &query, query_cb, NULL) != AEROSPIKE_OK)
        throwstream(runtime_error,
                    "aerospike_query_foreach() returned "
                    << err.code << '-' << err.message);

    as_query_destroy(&query);
}
```

使用 AQL 的 region-contains-points 查询示例。
- 将 point 数据插入到 aerospike。
```
aql> INSERT INTO test.testset (PK, geo_query_bin) VALUES (1, GEOJSON('{"type": "Polygon", "coordinates": [[[0,0], [0, 10], [10, 10], [10, 0], [0,0]]]}'))
```

- 查询该区域中的一个点。
```
AQL> SELECT * FROM test.testset WHERE geo_query_bin CONTAINS GeoJSON('{"type":"Point", "coordinates": [1, 1]}')
+---------------------------------------------------------------------------------------------+
| geo_query_bin                                                                               |
+---------------------------------------------------------------------------------------------+
| GeoJSON('{"type": "Polygon", "coordinates": [[[0,0], [0, 10], [10, 10], [10, 0], [0,0]]]}') |
+---------------------------------------------------------------------------------------------+
1 row in set (0.017 secs)

```

---

### <span id="query-filters"> Query Filters </span>

要扩展这两个查询的功能，请使用 UDF 来过滤结果集。

此示例 Python 脚本演示了如何使用过滤器 UDF。
```
def query_circle(args, client):
    """Query for records inside a circle."""
    query = client.query(args.nspace, args.set)
    predicate = aerospike.predicates.geo_within_radius(LOCBIN,
                                                       args.longitude,
                                                       args.latitude,
                                                       args.radius)
    query.where(predicate)

    # Search with UDF amenity filter
    query.apply('filter_by_amenity', 'apply_filter', [args.amenity,])
    query.foreach(print_value)
```

其中 `apply_filter` Lua 函数如下：
```
local function select_value(rec)
  return rec.val
end

function apply_filter(stream, amen)
  local function match_amenity(rec)
    return rec.map.amenity and rec.map.amenity == amen
  end
  return stream : filter(match_amenity) : map(select_value)
end
```

---

### <span id="index-on-list-map"> Index on list/map </span>

还可以使用 GeoJSON 数据类型对 list/map 元素进行索引和查询 -
```
    # create a secondary index for numeric values of test.demo records whose 'points' bin is a list of GeoJSON points
    client.index_list_create('test', 'demo', 'points', aerospike.INDEX_GEO2DSPHERE, 'demo_point_nidx')

    predicate = aerospike.predicates.geo_within_radius('points',
                                                       args.longitude,
                                                       args.latitude,
                                                       args.radius,
                                                       aerospike.INDEX_GEO2DSPHERE)

    query = client.query('test', 'demo')
    query.where(predicate);
```

以上将在 geoJSON list 元素上创建一个索引，并使用该索引构造查询 predicate。

---

### <span id="aerospike-geojson-extension"> Aerospike GeoJSON Extension </span>

使用 Aerospike `AeroCircle` 集合对象来存储圆形和正多边形。

此示例脚本在经度/维度 -122.250629, 37.871022 指定一个半径为 300 米的圆。
```
{"type": "AeroCircle", "coordinates": [[-122.250629, 37.871022], 300]}
```

---

### <span id="geojson-parsing"> GeoJSON Parsing </span>

在数据插入/更新时，Aerospike仅识别 `Point`, `Polygon`, `MultiPolygon`, and `AeroCircle` [GeoJSON geometry objects](http://geojson.org/geojson-spec.html#geometry-objects) ，它们是可索引对象。不受支持的 GeoJSON 对象返回 `AS_PROTO_RESULT_FAIL_GEO_INVALID_GEOJSON` 错误（例如，插入时 `LineString` 或 `MultiLineString` 失败）。根据 [GeoJSON Format Specification](http://geojson.org/geojson-spec.html#polygon) ，holes 可以是 `Polygon` objects。

`Polygon` 循环定义必须 wind counter-clockwise 。

Aerospike 支持 `Feature` 运算符，它允许对几何对象和用户指定的属性进行分组；但是，不支持 `Feature Collection` 。

插入/更新时捕获无效的 GeoJSON 对象。例如，定义为 `point` 而不是 `Point` 的对象失败。

根据 GeoJSON IETF 的建议，坐标系是 WGS84。 CRS 的显式规范被忽略。

---

### <span id="create-a-geospatial-application"> Create a Geospatial Application </span>

要开发地理空间应用程序：

1. [Install](https://docs.aerospike.com/docs/operations/install/index.html) 并配置 [Aerospike server](https://www.aerospike.com/download) 。
2. 在 namespace-set-bin 组合上创建 Geo2DSphere 索引。
3. 构建和插入 GeoJSON `Point` 数据。
4. 构造一个 `Points-in-Region` predicate (`where` clause)，发出查询请求，并处理返回的记录。
   1. （替代）构造和插入 GeoJSON `Polygon`/`MultiPolygon` 数据。
   2. （替代）构造一个 Region-contains-Point predicate，发出查询请求，并处理返回的记录。
  
---

### <span id="known-limitations"> Known Limitations </span>

- 不支持使用 UDF 插入/更新 GeoJSON 数据类型。
- 可以返回重复的记录。
- 对于 `data-in-memory` 为 true 的命名空间，GeoJSON 粒子(particle)最多分配比报告的粒子大小多 2KB，这在某些情况下会导致高内存消耗。此问题已在 Aerospike Server 版本 4.9.0+、4.8.0.8、4.7.0.12、4.6.0.14、4.5.3.16、4.5.2.16、4.5.1.21、4.5.0.24 中得到纠正。