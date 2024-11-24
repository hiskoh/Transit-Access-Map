# データ整備工程のまとめ

本ダッシュボードのデータソースとして、以下の3つのファイルを作成しました。

| データ                               | 概要                                                 | 出典・備考                                                                                      | ファイル名                  |
|--------------------------------------|------------------------------------------------------|-------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| **山口県名所情報**     | 山口市内の観光地等の目的地となる場所のデータ                                          | 山口県オープンデータカタログサイトや国土数値情報のデータを基に整備<br> ファイル名に「山口県」とありますが、実際には山口市の観光地情報のみを採録。 |[山口県名所_bounding_box.geojson](https://github.com/hiskoh/Transit-Access-Map/blob/96fae34d27b5708ad5ea05b8e6e998a27ea20580/dataset/%E5%B1%B1%E5%8F%A3%E7%9C%8C%E5%90%8D%E6%89%80_bounding_box.geojson)| 
| **公共交通拠点データ** |  公共交通の拠点（バス停・鉄道駅）の緯度経度情報（ポイントデータ）を採録したデータ        | 国土数値情報のデータを基に整備<br>Tableau内での関数処理の都合で、あえてCSV形式で整備。 |[公共交通拠点.csv](https://github.com/hiskoh/Transit-Access-Map/blob/96fae34d27b5708ad5ea05b8e6e998a27ea20580/dataset/%E5%85%AC%E5%85%B1%E4%BA%A4%E9%80%9A%E6%8B%A0%E7%82%B9.csv)         |
| **公共交通路線データ** |公共交通の路線（バス路線・鉄道路線）の緯度経度情報（ラインデータ）を採録したデータ        | 国土数値情報のデータを基に整備                                                                 | [公共交通路線.geojson](https://github.com/hiskoh/Transit-Access-Map/blob/96fae34d27b5708ad5ea05b8e6e998a27ea20580/dataset/%E5%85%AC%E5%85%B1%E4%BA%A4%E9%80%9A%E8%B7%AF%E7%B7%9A.zip)     | 

なお、使用したデータの詳細については、[使用したデータセット](https://github.com/hiskoh/Transit-Access-Map/blob/main/README.md#%E4%BD%BF%E7%94%A8%E3%81%97%E3%81%9F%E3%83%87%E3%83%BC%E3%82%BF%E3%82%BB%E3%83%83%E3%83%88)を参照してください。  

---

## 1. 山口県名所情報

ここでのアウトプット： [山口県名所_bounding_box.geojson](https://github.com/hiskoh/Transit-Access-Map/blob/96fae34d27b5708ad5ea05b8e6e998a27ea20580/dataset/%E5%B1%B1%E5%8F%A3%E7%9C%8C%E5%90%8D%E6%89%80_bounding_box.geojson)

1. **観光地データの収集**
   
   以下のデータを使用し、観光地情報を収集しました
   * 山口県 観光施設一覧  
   * 山口市 観光交流施設  
   * 観光資源データ  
   * 景観重要建造物・樹木データ  
   * 地域資源データ  
   * 都道府県指定文化財データ  
   * 山口市 公園・遊び場  

3. **Googleスプレッドシートでデータ統合**
   
   上記データを [Googleスプレッドシート](https://docs.google.com/spreadsheets/d/1NFaXy6OWznxPCCeqg_-LTzas5_n4faKJERXvdGIFPEo/edit?usp=sharing)に集約し、データを整理しました。施設情報の英訳は `=GOOGLETRANSLATE()` を用いています。

4. **PostGISでデータ処理**
   
   山口市分のデータをPostgreSQLに格納し、ダッシュボードで公共施設との紐付けを行うための半径1,000m内バウンディングボックスを生成。  

   ```sql
   CREATE TABLE yamaguchi_poi_bounding_box AS
   SELECT 
       "No.", "名称", "名称_英語（Google自動翻訳）", "種別", "説明", "出典", "ライセンス", "時点",
       ST_X(geom) AS lon, ST_Y(geom) AS lat,
       ST_Envelope(ST_Buffer(geom::geography, 1000)::geometry) AS bounding_box_geom
   FROM yamaguchi_poi;
   ```

5. **GeoJSON形式でエクスポート**
   
   上記テーブルをQGISでGeoJSON形式で出力。

## 2. 公共交通拠点データ

ここでのアウトプット： [公共交通拠点.csv](https://github.com/hiskoh/Transit-Access-Map/blob/96fae34d27b5708ad5ea05b8e6e998a27ea20580/dataset/%E5%85%AC%E5%85%B1%E4%BA%A4%E9%80%9A%E6%8B%A0%E7%82%B9.csv)

1. **国土数値情報のバス停データ、鉄道駅データをPostgreSQLのDBに取り込み**

   バス停、鉄道駅データをそれぞれ以下の名称で取り込みました。
     - `transportation.bus`
     - `transportation.station`

   また、山口県内の公共交通拠点に絞り込むため、国土数値情報行政区域データを`city_div.div01_pref`として取り込んでいます（都道府県名のカラムを`都道府県`という日本語カラムで取り込んでいます）

3. **データの加工**
  
   鉄道駅データとバス停データを統合し、1つのテーブルにまとめます。統合処理のクエリは[hoge](hoge)に記載しました。
   - 鉄道駅データについて、元データはラインデータですが、バス停データと統合できるようにポイントデータに変換しています。 
   - バス停データについて、国土数値情報の元データでは1つのカラムに複数バス路線が採録されている場合があるため、区切り文字（`,` `:` `/`）で分割して別レコードにする工夫を施しました。

  
   ```sql
   CREATE TABLE transportation."停留所・鉄道駅" AS
   WITH combined_data AS (
    SELECT 
        '鉄道' AS "種別",
        "N02_004" AS "企業名",
        "N02_003" AS "路線名",
        "N02_005" AS "拠点名",
        ST_PointOnSurface(geom) as geom
    FROM transportation.station
    UNION ALL
    SELECT 
        'バス' AS "種別",
        "P11_002" AS "企業名",
        regexp_split_to_table(unnest(ARRAY[
            "P11_003_01", "P11_003_02", "P11_003_03", "P11_003_04", "P11_003_05", 
            "P11_003_06", "P11_003_07", "P11_003_08", "P11_003_09", "P11_003_10", 
            "P11_003_11", "P11_003_12", "P11_003_13", "P11_003_14", "P11_003_15", 
            "P11_003_16", "P11_003_17", "P11_003_18", "P11_003_19", "P11_003_20", 
            "P11_003_21", "P11_003_22", "P11_003_23", "P11_003_24", "P11_003_25", 
            "P11_003_26", "P11_003_27", "P11_003_28", "P11_003_29", "P11_003_30", 
            "P11_003_31", "P11_003_32", "P11_003_33", "P11_003_34", "P11_003_35"
        ]), ',|・|/') AS "路線名",
        "P11_001" AS "拠点名",
        geom
    FROM transportation.bus
   )
   SELECT 
    ROW_NUMBER() OVER () AS "識別番号",
    "種別", "企業名", "路線名", "拠点名", combined_data.geom
   FROM combined_data,city_div.div01_pref
   WHERE "路線名" IS NOT NULL
     AND "都道府県" = '山口県'
     AND ST_Intersects(combined_data.geom,div01_pref.geom)
   ```

3. **CSVで出力**

   上記テーブルをCSV形式でエクスポートします。

## 公共交通路線データ

   ここでのアウトプット： [公共交通路線.geojson](https://github.com/hiskoh/Transit-Access-Map/blob/96fae34d27b5708ad5ea05b8e6e998a27ea20580/dataset/%E5%85%AC%E5%85%B1%E4%BA%A4%E9%80%9A%E8%B7%AF%E7%B7%9A.zip)

   **背景:**  
   国土数値情報のバスルートデータには路線情報が存在しないため、バス停情報を基に路線を推定しました。この作業が結構厄介で、実は本ダッシュボードを作る中で一番手間がかかった部分です…。

1. **国土数値情報のバスルートデータ、鉄道路線データをPostgreSQLに取り込み**
  
      バス路線、鉄道路線データをそれぞれ以下の名称で取り込みました。
   
     - `transportation."N07-22_35"` …バス路線
     - `transportation."N02-23_RailroadSection"` …鉄道路線
 
2. **PostGISでデータ統合と推定処理**

   バスルートデータには路線情報がないため、バス停情報を基に路線名を推定する処理をPostGISで実行。

   ```sql
   -- 1. 停留所テーブルにバッファカラムが存在しない場合は追加
   ALTER TABLE transportation."停留所・鉄道駅" ADD COLUMN IF NOT EXISTS buffer10m geometry(Polygon, 4326);

   -- 2. 停留所に10メートルバッファを事前に作成
   UPDATE transportation."停留所・鉄道駅"
   SET buffer10m = ST_Buffer(geom::geography, 10)::geometry
   WHERE buffer10m IS NULL;

   -- 3. インデックスを作成して高速化
   CREATE INDEX IF NOT EXISTS idx_stops_geom ON transportation."停留所・鉄道駅" USING GIST (geom);
   CREATE INDEX IF NOT EXISTS idx_stops_buffer10m ON transportation."停留所・鉄道駅" USING GIST (buffer10m);

   -- 4. バスルートの分割済みラインを作成
   DROP TABLE IF EXISTS transportation.split_route;

   CREATE TABLE transportation.split_route AS
   WITH intersections AS (
    SELECT 
        a."N07_001" as "企業",
        ST_Intersection(a.geom, b.geom) AS intersection_geom,
        a.geom AS original_geom
    FROM transportation."N07-22_35" a, transportation."N07-22_35" b
    WHERE ST_Intersects(a.geom, b.geom)
   ),
   split_lines AS (
    SELECT DISTINCT
	    s_stops."識別番号",
        intersections."企業",
	    s_stops."路線名",
		split.geom AS geom,
        -- バス停がある方をstart_bufferとして設定
        CASE 
            WHEN ST_Within(s_stops.geom, ST_Buffer(ST_StartPoint(split.geom)::geography, 10)::geometry) 
                THEN ST_Buffer(ST_StartPoint(split.geom)::geography, 10)::geometry 
            ELSE ST_Buffer(ST_EndPoint(split.geom)::geography, 10)::geometry 
        END AS start_buffer,
        CASE 
            WHEN ST_Within(s_stops.geom, ST_Buffer(ST_StartPoint(split.geom)::geography, 10)::geometry) 
                THEN ST_Buffer(ST_EndPoint(split.geom)::geography, 10)::geometry 
            ELSE ST_Buffer(ST_StartPoint(split.geom)::geography, 10)::geometry 
        END AS end_buffer
    FROM 
        intersections
	JOIN LATERAL ST_Dump(ST_Split(intersections.original_geom, intersections.intersection_geom)) AS split ON true
    LEFT JOIN transportation."停留所・鉄道駅" s_stops 
        ON ST_Intersects(split.geom, s_stops.buffer10m) and intersections."企業" = s_stops."企業名"  -- バス停が近い側を判別
    WHERE 
        ST_GeometryType(intersection_geom) IN ('ST_Point', 'ST_MultiPoint') 
        AND ST_GeometryType(split.geom) = 'ST_LineString'
        AND ST_Length(split.geom) > 0
   )
   SELECT * FROM split_lines;

   -- 分割ラインに対するインデックス作成
   CREATE INDEX IF NOT EXISTS idx_split_geom ON transportation.split_route USING GIST (geom);
   CREATE INDEX IF NOT EXISTS idx_split_start_buffer ON transportation.split_route USING GIST (start_buffer);
   CREATE INDEX IF NOT EXISTS idx_split_end_buffer ON transportation.split_route USING GIST (end_buffer);

   -- 5. 路線経路生成の最適化クエリ
   DO $$
   DECLARE
    i INT := 1;
    max_iterations INT := 3;
   BEGIN
    -- 初回テーブルを作成
    DROP TABLE IF EXISTS transportation.tmp_all_stops;
    CREATE TABLE transportation.tmp_all_stops AS
    SELECT DISTINCT
        sl."識別番号",
        sl."企業",
        sl."路線名",
        sl.geom,
        sl.start_buffer,
        sl.end_buffer,
        true AS start_stop,
        CASE WHEN sl."路線名" = e_stops."路線名" THEN true else false end AS end_stop,
        1 AS counter
    FROM transportation.split_route sl
    LEFT JOIN transportation."停留所・鉄道駅" e_stops ON ST_Within(e_stops.geom, sl.end_buffer) and sl."企業" = e_stops."企業名" and sl."路線名" = e_stops."路線名"
	WHERE sl."路線名" is not null;
   --接続済、未接続を分離	
    DROP TABLE IF EXISTS transportation.tmp_connected_stops ;
    CREATE TABLE transportation.tmp_connected_stops AS SELECT * from transportation.tmp_all_stops where end_stop is true;
	
    DROP TABLE IF EXISTS transportation.tmp_single_side_stops  ;
    CREATE TABLE transportation.tmp_single_side_stops  AS SELECT * from transportation.tmp_all_stops where end_stop is not true;
    CREATE INDEX idx_single_side_stops_geom ON transportation.tmp_single_side_stops  USING GIST (geom);
    CREATE INDEX idx_single_side_stops_start_buffer ON transportation.tmp_single_side_stops  USING GIST (start_buffer);
    CREATE INDEX idx_single_side_stops_end_buffer ON transportation.tmp_single_side_stops  USING GIST (end_buffer);
    -- ループ処理でテーブルを更新していく
    FOR i IN 2..50 LOOP
        -- 新しい行を一時テーブル tmp_new に追加
        DROP TABLE IF EXISTS transportation.tmp_new_connected_stops;
        CREATE  TABLE transportation.tmp_new_connected_stops AS
        with tmp as (
		SELECT
            rp."識別番号",
            rp."企業",
            rp."路線名",
            ST_LineMerge(ST_Union(rp.geom, sl.geom)) AS geom,
			rp.end_buffer rpe,
			sl.start_buffer sls,
		 	sl.end_buffer sle,
            rp.start_buffer,
            CASE
                WHEN ST_Intersects(rp.end_buffer, sl.end_buffer) THEN sl.start_buffer
			    ELSE sl.end_buffer
            END AS end_buffer,
            true AS start_stop,
            CASE WHEN rp."路線名" = e_stops."路線名" THEN true else false end AS end_stop,
            rp.counter + 1 AS counter,
			rp.end_buffer as old_end_buffer
        FROM transportation.tmp_single_side_stops rp
        JOIN
            transportation.split_route sl
            ON rp."企業" = sl."企業"
            AND ST_Intersects(rp.end_buffer, sl.geom)
            AND Not ST_Equals(rp.geom, sl.geom)
        LEFT JOIN transportation."停留所・鉄道駅" e_stops ON (ST_Within(e_stops.geom, sl.start_buffer) OR ST_Within(e_stops.geom, sl.end_buffer)) and rp."路線名" = e_stops."路線名"
		)
		SELECT distinct on("識別番号",old_end_buffer,end_buffer)
		"識別番号","企業","路線名",geom,start_buffer,end_buffer, start_stop, end_stop, counter
		from tmp
		ORDER BY  "識別番号",old_end_buffer,end_buffer,end_stop desc
		;
		
        -- 新しい行を tmp_connected_stops に挿入
        INSERT INTO transportation.tmp_connected_stops
        SELECT * FROM transportation.tmp_new_connected_stops where end_stop is true;
        -- tmp_single_side_stops を更新
        Drop table transportation.tmp_single_side_stops;
		Create table transportation.tmp_single_side_stops as
		SELECT * FROM transportation.tmp_new_connected_stops where end_stop is not true and "識別番号" not in (SELECT "識別番号" from transportation.tmp_connected_stops);
        CREATE INDEX idx_single_side_stops_geom ON transportation.tmp_single_side_stops  USING GIST (geom);
        CREATE INDEX idx_single_side_stops_start_buffer ON transportation.tmp_single_side_stops  USING GIST (start_buffer);
        CREATE INDEX idx_single_side_stops_end_buffer ON transportation.tmp_single_side_stops  USING GIST (end_buffer);
		
        -- カウンタと条件を使ってループ終了を判断
        IF (SELECT COUNT(*) FROM transportation.tmp_single_side_stops) = 0 THEN
            EXIT;
        END IF;
    END LOOP;

   Drop table if exists transportation.connected_stops;
   Create table transportation.connected_stops as
   SELECT DISTINCT
    "企業",
    "路線名",
    ST_GeometryN((ST_Dump(ST_LineMerge(ST_Union(geom)))).geom, 1) AS geom
   FROM transportation.tmp_connected_stops
   GROUP BY "企業", "路線名";
   END $$;
   ```

3. **バス路線データと鉄道路線データの統合**

   直前に整備したバス路線データと鉄道路線データを同じテーブルに統合。
   なお鉄道路線データは全国分存在するので、統合の際に山口県を通る路線に絞り込み。

   ```sql
   CREATE TABLE  transportation.line_marge_data as
   SELECT "企業", "路線名", geom
   FROM transportation.tmp_connected_stops
   UNION ALL
   SELECT n02_004 as "企業名",n02_003 as "路線名", "N02-23_RailroadSection".geom
   FROM transportation."N02-23_RailroadSection",city_div.div01_pref
   WHERE "都道府県" = '山口県'
     AND ST_Intersects("N02-23_RailroadSection".geom,div01_pref.geom)
   ```
    
4. **GeoJSONで出力**

   上記テーブルをGeoJSON形式でエクスポートします。

## まとめ

   データ整備に想定以上に時間を要してしまいましたが、オープンデータ等の活用事例として何かの参考になれば幸いです。
