# heian-kyo_excavation
「平安京跡データベース」（※1）によって公開されているオープンデータを用いた分析を行ったコード。

武内樹治・今村聡・矢野桂司 2021「「平安京跡データベース」の利活用に向けた課題とその検証」（※2）にて行った平安京跡データベースを用いた分析（第3章にあたる）のRのコードを示す。なお、当論文では、地図の可視化にESRI社のArcMapを用いているため、論文内の図とRで可視化した図は表示の仕方が異なっている。
データや分析内容については、当文献を参考にされたい。


平安京跡での発掘調査を集成した平安京跡データーベースを用いて発掘調査地点を時空間的に分析することでどのようなことが見えてくるのかを示し、当資料や発掘調査の情報の活用可能性を指摘することを目指している。


※1平安京跡データーベース（https://heiankyoexcavationdb-rstgis.hub.arcgis.com/）

※2武内樹治・今村聡・矢野桂司 2021「「平安京跡データベース」の利活用に向けた課題とその検証」『アート・リサーチ』21, pp.71-81
https://ritsumei.repo.nii.ac.jp/?action=pages_view_main&active_action=repository_view_main_item_detail&item_id=14498&item_no=1&page_id=13&block_id=21




##分析の準備
```{r}
# パッケージ読み込み
library(tidyverse)
library(sf)
library(ggthemes)
library(viridis)
library(lwgeom)
library(GISTools)
library(raster)

#データ読み込み
#「平安京跡データーベース」より、平安京のポリゴンと発掘調査地点のポイントデータを入手し、読み込む。
発掘調査point_20211205:平安京跡で行われた発掘調査の集成（https://heiankyoexcavationdb-rstgis.hub.arcgis.com/datasets/f222501ec417445e872124a0f5d80125_0/about）
#平安京ポリゴン：平安京跡の各条坊を考慮したポリゴンデータ（https://heiankyoexcavationdb-rstgis.hub.arcgis.com/datasets/5672376f6bc64d0e9ac6eac0e6596ff8_1/about）
#heiankyo_area:平安京跡の範囲のポリゴンデータ（https://heiankyoexcavationdb-rstgis.hub.arcgis.com/datasets/b264882862ec4a9b8ed509dd3304655b_0/about）

heiankyo_poly <- st_read("平安京ポリゴン.shp")
heiankyo_excavation <- st_read("発掘調査point_20211205.shp")
heiankyo_area <- st_read("heiankyo_area.shp")
```


##分析の前処理
```{r}
#座標系変換
heiankyo_poly_2448 <- heiankyo_poly %>% st_transform(2448)
st_crs(heiankyo_poly_2448)
heiankyo_excavation_2448 <- heiankyo_excavation %>% st_transform(2448)
st_crs(heiankyo_excavation_2448)
heiankyo_area_2448 <- heiankyo_area %>% st_transform(2448)

#もとの座標系での表示
library(ggplot2)
ggplot()+
  geom_sf(data = heiankyo_poly)+
  geom_sf(data = heiankyo_excavation)+
  theme_map()

#平面直角座標系での地図化
ggplot()+
  geom_sf(data = heiankyo_poly_2448)+
  geom_sf(data = heiankyo_excavation_2448)+
  theme_map()

ggplot()+
  geom_sf(data = heiankyo_area_2448)+
  geom_sf(data = heiankyo_excavation_2448)+
  theme_map()

#発掘調査ポイントのデータが平安京跡の外のデータも一部入っているため、平安京跡内だけにする
heiankyo_excavation_area <- st_intersection(x = heiankyo_area_2448, y = heiankyo_excavation_2448)

#平安京跡内だけのデータを再表示
ggplot()+
  geom_sf(data = heiankyo_poly_2448)+
  geom_sf(data = heiankyo_excavation_area)+
  theme_map()


#調査年代ごとに分ける
#属性の西暦年を数値へ変換
class(heiankyo_excavation_area$西暦年)
heiankyo_excavation_area$year <- as.numeric(heiankyo_excavation_area$西暦年)
class(heiankyo_excavation_area$year)

#西暦年が1970~1999年のものを対象に、1970,1980,1990年代ごとに分ける
heiankyo_excavation_area_1970s <- filter(heiankyo_excavation_area, between(heiankyo_excavation_area$year,1970,1979))
heiankyo_excavation_area_1980s <- filter(heiankyo_excavation_area, between(heiankyo_excavation_area$year,1980,1989))
heiankyo_excavation_area_1990s <- filter(heiankyo_excavation_area, between(heiankyo_excavation_area$year,1990,1999))
```

![image](https://user-images.githubusercontent.com/54834814/169643973-6f583005-19c4-4bbc-9324-5ee33e775d7e.png)
平安京跡内だけのデータを再表示したもの


##年代ごとに分けた発掘調査地点の密度分析
#1970s
```{r}
ggplot()+
  geom_sf(data = heiankyo_poly_2448)+
  geom_sf(data = heiankyo_excavation_area_1970s)+
  theme_map()

#カーネル密度
a1970s.dens <- heiankyo_excavation_area_1970s %>% as("Spatial") %>% 
  kde.points(n=100, h = 500)
a1970s.dens_d <- a1970s.dens$kde %>% as.data.frame()
a1970s.dens_x <- a1970s.dens$Var1 %>% as.data.frame()
a1970s.dens_y <- a1970s.dens$Var2 %>% as.data.frame()
a1970s.dens_data <- bind_cols(a1970s.dens_d, a1970s.dens_x, a1970s.dens_y)
colnames(a1970s.dens_data) <- c("dens", "x", "y")

#カーネル密度結果をオーバーレイ
a1970s.dens_data %>% 
  ggplot()+
  geom_raster(aes(x=x, y=y, fill=dens))+
  geom_sf(data = heiankyo_poly_2448, size = 0.05, fill = NA)+
  geom_sf(data = heiankyo_excavation_area_1970s, size = 1.0, colour = "Red")+
  geom_contour(
    aes(x=x,y=y,z=dens),
    size = 0.3, colour = "White",
  )+
  scale_fill_viridis()+
  theme_minimal()+
  theme(
    axis.title = element_blank()
  )

```

![image](https://user-images.githubusercontent.com/54834814/169643862-ca2f8d99-7cba-4a5f-b28c-0786e757bf93.png)

#1980s
```{r}
ggplot()+
  geom_sf(data = heiankyo_poly_2448)+
  geom_sf(data = heiankyo_excavation_area_1980s)+
  theme_map()

#カーネル密度
a1980s.dens <- heiankyo_excavation_area_1980s %>% as("Spatial") %>% 
  kde.points(n=100, h = 500)
a1980s.dens_d <- a1980s.dens$kde %>% as.data.frame()
a1980s.dens_x <- a1980s.dens$Var1 %>% as.data.frame()
a1980s.dens_y <- a1980s.dens$Var2 %>% as.data.frame()
a1980s.dens_data <- bind_cols(a1980s.dens_d, a1980s.dens_x, a1980s.dens_y)
colnames(a1980s.dens_data) <- c("dens", "x", "y")

#カーネル密度結果をオーバーレイ
a1980s.dens_data %>% 
  ggplot()+
  geom_raster(aes(x=x, y=y, fill=dens))+
  geom_sf(data = heiankyo_poly_2448, size = 0.05, fill = NA)+
  geom_sf(data = heiankyo_excavation_area_1980s, size = 1.0, colour = "White")+
  geom_contour(
    aes(x=x,y=y,z=dens),
    size = 0.3, colour = "White",
  )+
  scale_fill_viridis()+
  theme_minimal()+
  theme(
    axis.title = element_blank()
  )
  
```
  ![image](https://user-images.githubusercontent.com/54834814/169643876-06a6911c-a506-4ed7-8f16-ff691f42512a.png)
  
#1990s
```{r}
ggplot()+
  geom_sf(data = heiankyo_poly_2448)+
  geom_sf(data = heiankyo_excavation_area_1990s)+
  theme_map()

#カーネル密度
a1990s.dens <- heiankyo_excavation_area_1990s %>% as("Spatial") %>% 
  kde.points(n=100, h = 500)
a1990s.dens_d <- a1990s.dens$kde %>% as.data.frame()
a1990s.dens_x <- a1990s.dens$Var1 %>% as.data.frame()
a1990s.dens_y <- a1990s.dens$Var2 %>% as.data.frame()
a1990s.dens_data <- bind_cols(a1990s.dens_d, a1990s.dens_x, a1990s.dens_y)
colnames(a1990s.dens_data) <- c("dens", "x", "y")

#カーネル密度結果をオーバーレイ
a1990s.dens_data %>% 
  ggplot()+
  geom_raster(aes(x=x, y=y, fill=dens))+
  geom_sf(data = heiankyo_poly_2448, size = 0.05, fill = NA)+
  geom_sf(data = heiankyo_excavation_area_1990s, size = 1.0, colour = "Red")+
  geom_contour(
    aes(x=x,y=y,z=dens),
    size = 0.3, colour = "White",
  )+
  scale_fill_viridis()+
  theme_minimal()+
  theme(
    axis.title = element_blank()
  )


```
  ![image](https://user-images.githubusercontent.com/54834814/169643891-445b77b2-6939-45d5-83ee-b5754c2fc501.png)
