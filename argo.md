#### `self.city_lane_centerlines_dict`

`=self.build_centerline_index()`

`dict`：`city_name：lane_objs`

为每个城市建立一个字典，保存从`map_files/pruned_argoverse_PIT_10314_vector_map.xml`读取的该城市所有道路的信息，字典的`key：laneid`，`value:`记录道路不同属性值的`LaneSegment`对象。

`xml`文件解读：

根元素下有两种元素：`node`和`way`

`tag=node`记录城市所有`centerline`的坐标值
![image](https://user-images.githubusercontent.com/95835767/145328160-e20d3868-6219-4655-9284-1b184a1092dc.png)

将`node`元素的属性值读取到字典`all_gragh_nodes`中，保存全部`centerline`结点信息（x,y,id,height）

`tag=way`记录每条lane的信息，‘has_traffic_control’:是否有交通管制，‘turn_direction’:转向，‘is_intersection’:是否为路口，‘l_neighbor_id’,'r_neighbor_id':左右邻居，‘predecessor’，‘successor’:前驱后继；‘ref'记录该条lane的所有centerlines，属性值对应于`tag=node`中的id。

![image](https://user-images.githubusercontent.com/95835767/145328172-76c5033a-b727-4306-8e66-cbc0854cda3b.png)

遍历元素way中的信息，将属性名为`k`的信息存入`lane_obj`中，将属性名为`ref`的属性值存入`node_id_list`中

ref对应`all_gragh_nodes`中的id,保存过程会通过`convert_node_id_list_to_xy`转换为x,y（height）值

将`xml`文件中的属性值读取到字典中，通过`convert_dictionary_to_lane_segment_obj`转为`lane_segment`对象

返回值：`lane_objs`：dict,`lane_id:lane_segment`

`PS`:`xml`文件——

![image](https://user-images.githubusercontent.com/95835767/145328185-8f319d04-86c0-4e3d-b904-720e9b38e851.png)

根元素（有且只有一个）\<`ArgoverseVectorMap`>

其他元素\<node>

\<元素名 属性名 = “属性值”  属性名 = “属性值”>

\<node id="0" x="3168">



#### `self.city_halluc_bbox_table`

#### `self.city_halluc_tableidx_to_laneid_map`

城市道路的车道区域矩形坐标

`=build_hallucinated_lane_bbox_index`

每一条车道对应一个车道区域的多边形

`PIT_10314_tableidx_to_laneid_map.json` `i`到`land_id`的映射

`PIT_10314_halluc_bbox_table.npy`：对于每条lane，包裹多边形区域的最小矩形的左上角与右下角的坐标值`lane_num*4`

返回值：

`city_halluc_tableidx_to_laneid_map`：表示`lane_id`映射关系的字典 => `self.city_halluc_tableidx_to_laneid_map`

` city_halluc_bbox_table`：`dict`,`city_name`:[`array`, 第`i`个元素是对应上述映射关系后的`lane_id`的矩形坐标] => `self.city_halluc_bbox_table`



#### `self.city_rasterized_da_roi_dict`

城市的可驾驶区域

`build_city_driveable_area_roi_index()`

`PIT_10314_driveable_area_mat_2019_05_28.npy`：一个城市的可驾驶区域，使用栅格化地图表示，分割率为1米，用0表示不可驾驶区域，用1表示可驾驶区域。

`PIT_10314_npyimage_to_city_se2_2019_05_28.npy`：`pkl`图像坐标到城市坐标的转换矩阵。

返回值：
`city_rasterized_da_roi_dict`:

​	`dict,{city_name:{da_mat,npyimage_to_city_se2,roi_mat}}`

​		`da_mat`：表示城市可行驶区域的二进制值（`1m`网格分辨率），0代表不可行驶，1代表可行驶

​		`	city_to_pkl_image_se2`：将`pkl`图像中的点转换为城市坐标，是一个`3*3`的array。注：真实坐标为`(x,y)`的点，在图片中坐标为`(y,x)`。

​		`roi_mat`：`cv2.distanceTransform`计算图像每个像素点到最近零像素的距离。区分前景值（1）与背景值（0）



#### `self.city_rasterized_ground_height_dict`

`=build_city_ground_height_index()`

城市地面高度值

`PIT_10314_ground_height_mat_2019_05_28.npy`光栅化地图表示1米分辨率的城市地面高度

返回值：

​	——字典 `city_name`：`dict`

​            `ground_height`:`1m`网格分辨率的实时地面高度

​			`npyimage_to_city_se2`：将`pkl`图像中的点转换为城市坐标的转换矩阵

#### `self.city_to_lane_polygons_dict`

对于每条车道，将中心线转换为表示车道区域的多边形，使用平均车道宽度3.8（centerline左右两边分别1.9）

`self.get_lane_segment_polygon()`->`centerline_to_polygon()`

通过计算斜率的负倒数计算夹角，计算3.8米的车道宽度对应centerline`（N，2）`左右车道边缘的坐标:`left_centerline:(N,2)`,`right_centerline(N,2)`

`convert_lane_boundaries_to_polygon()`

顺序连接右边缘`right_centerline`和左边缘`left_centerline`，返回形状为`(2N+1,2)`的数组，第一个顶点和最后一个顶点重复。

加中心点的高度

`pts_z = self.get_ground_height_at_xy()`

将`(2N+1,2)`个多边形边界坐标通过`SE2`坐标转换为城市坐标，通过在已构建的光栅图中读取的地面高度，查找多边形边缘坐标分别对应的高度值，添加到相应的多边形边界坐标中，形成`(2N+1,3)`大小的数组。（连接的3维前两维是`pkl`图像坐标）

返回值：

多维数组，每一个元素对应第几条路的`（2N+1，2）`维的多边形边框坐标。

#### `self.city_to_driveable_areas_dict`

`self.get_vector_map_driveable_areas()`

根据城市的二进制可驾驶区域` self.city_rasterized_da_roi_dict[city_name]["da_mat"]`使用`OpenCV`的方法获得轮廓线。

`get_da_contours()`:

`cv2.threshold(src,thesh,maxval,type[,dst])`阈值处理，将灰度图像处理为非黑即白的二值图像。

这个函数有四个参数

1. `src`：图片源

2. `thresh`：阈值

3. `maxval`：高于（低于）阈值时赋的新值

4. `type`：使用什么类型的算法，常用的有：
   •`cv2.THRESH_BINARY（`黑白二值）
   •` cv2.THRESH_BINARY_INV`（黑白二值反转）
   •` cv2.THRESH_TRUNC `（得到的图像为多像素值）
   •`cv2.THRESH_TOZERO`
   •`cv2.THRESH_TOZERO_INV`

![image](https://user-images.githubusercontent.com/95835767/145328237-19416af4-f3d6-4e89-a857-facaf588def7.png)
`contours,hierarchy = cv2.findContours(image,mode,method)`

mode:轮廓检索模式

1. `cv2.RETR_EXTERNAL`    表示只检测外轮廓

2. `cv2.RETR_LIST `              检测的轮廓不建立等级关系
3. `cv2.RETR_CCOMP `         建立两个等级的轮廓，上面的一层为外边界，里面的一层为内孔的边界信息。如果内孔内还有一个连通物体，这个物体的边界也在顶层。
4. `cv2.RETR_TREE`            建立一个等级树结构的轮廓。

method:轮廓近似办法

1. `cv2.CHAIN_APPROX_NONE `存储所有的轮廓点，相邻的两个点的像素位置差不超过1，即`max（abs（x1-x2），abs（y2-y1））==1`
2. `cv2.CHAIN_APPROX_SIMPLE` 压缩水平方向，垂直方向，对角线方向的元素，只保留该方向的终点坐标，例如一个矩形轮廓只需4个点来保存轮廓信息
3. `cv2.CHAIN_APPROX_TC89_L1，CV_CHAIN_APPROX_TC89_KCOS` 使用`teh-Chinl chain` 近似算法

返回值：

​	`cv2.findContours()`函数返回两个值，一个是轮廓本身，还有一个是每条轮廓对应的属性。

`contour` :一个list，list中每个元素都是图像中的一个轮廓，用`numpy`中的`ndarray`表示



得到的contour是城市坐标系（因为本来就是通过城市的可驾驶区域矩阵计算得到的），通过`SE2`变换矩阵转换为`pkl`图片坐标系，然后使用同样的方法为轮廓上的坐标点查询高度值，变为三维数组（前两维为变换后的`pkl`图片的坐标），返回`city_contours`，其中每一个元素都表示一条轮廓线的list。



#### `self.city_to_lane_bboxes_dict`

#### `self.city_to_da_bboxes_dict`

提供能够包围输入多边形的边界框的最小尺寸

`[x_min,y_min,x_max,y_max]`

return:`(n*4)`的浮点数组

比较得到和`self.city_halluc_bbox_table`是相同的。



#### `def get_lane_ids_in_xy_bbox`

给定一个query点（query_x,query_y）,查找距离query点曼哈顿距离为`query_search_range_manhattan`范围内的所有lane。

query范围：

```python
query_min_x = query_x - query_search_range_manhattan
query_max_x = query_x + query_search_range_manhattan
query_min_y = query_y - query_search_range_manhattan
query_max_y = query_y + query_search_range_manhattan
```



某lane满足在query范围内的条件——（1|2）&（3|4）：

1. 车道线矩形区域的最小x坐标`x1`在query范围里或车道线矩形区域的最大x坐标的`x2`在query范围里
2. `x1`和`x2`分别在query范围边界线的两边
3. 车道线矩形区域的最小y坐标`y1`在query范围里或车道线矩形区域的最大y坐标`y2`在query范围里
4. `y1`和`y2`分别在query范围边界线的上下

返回值：

方法`find_all_polygon_bboxes_overlapping_query_bbox`返回满足条件的道路对应的`0-lane_num`的索引

再根据索引到`lane_id`的映射关系（`self.city_halluc_tableidx_to_laneid_map`）返回查找到的满足条件的`lane_id`



#### `def remove_overlapping_lane_seq()`

去除重叠的车道集合序列：

1.  `s1------s2-----------------e1--------e2` lane2开始在lane1的中间某处，结束在lane1结束之后

   ```python
   lane_seq2[0] in lane_seq1[1:] and lane_seq1[-1] in lane_seq2[:-1]
   ```

   

2. `s1------s2-----------------e2--------e1`lane2开始在lane1中间某处，结束在lane1结束之前

   ```python
   set(lane_seq2) <= set(lane_seq1)
   ```

   

对得到的child集合（也即dfs搜到的车道序列）进行打分排序

1. 首先根据

2. 然后根据距离再次排序，`def get_heuristic_centerlines_for_test_set`

   `am.get_cl_from_lane_seq()`融合道路序列中所有道路的中心线

   `project`函数计算中心线集合构成的几何图形到agent_start以及agent_end的最短距离

   1. 如果`end_dist > start_dist`总体排在前边，它们之间还保持之前打分的排序

   2. 如果`end_dist < start_dist`总体排在后边，它们之间有新的打分概率，首先根据阈值和`end_dist > start_dist`个数确定个数，根据score计算概率，依照概率采样

#### `def remove_ground_surface()`

查询点云高度值，判断是否在高度阈值内，决定它是否是在平地上的点，保留非平地点并返回。



####  `def get_lane_segment_containing_xy()`

查询query所在的车道

首先查找query附近的车道，对所有的车道，看query是否包含在车道多边形内

#### `def get_raster_layer_points_boolean()`

判断某些位置是否是可驾驶区域

#### `def get_nearest_centerline()`

查找距离query最近的centerline

通过`get_lane_ids_in_xy_bbox（）`找一定范围的附近车道，如果找不到则将搜索曼哈顿距离增加两倍。

`lane_waypt_to_query_dist()`：

​	对每个附近车道，沿着centerline插入等距点，计算扩展后50个centerline到query的距离，取最小值

​	1. 将所有附近车道的centerline到query距离的最小值存入per_lane_dists返回

​	2. per_lane_dists从小到大排序的索引值存入 min_dist_nn_indices 返回

​	3. 所有车道的centerlines(扩展后50维的)存入`dense_ceenterlines `返回

返回值：

1. 距离最小的（min_dist_nn_indices[0]）的道路的laneSegment对象

2. 相应的 50 维 centerlines
3. 计算的置信度

#### `def get_lane_direction()`

获取所在车道的矢量方向。

首先获取距离query最近的centerline所在的车道，求50个centerline到query的距离，取最小距离的两个点的索引，返回这两个点中较大点与较小点的centerline坐标差值与置信度

#### `def get_lane_segment_containing_xy()`

给定query，列举所有车道的多边形区域包含query点的车道

先查找一定区域内的附近车道，然后获取多边形，遍历query是否在多边形内。

#### `def remove_extended_control_measure()`

如果某种关于query的车道查找，从车道序列中查找到了太多的前辈，则删除的操作

如果车道序列中的车道是query的占用车道（见get_lane_segment_containing_xy）,则忽略车道之前出现的所有车道。

`def interp_arc()`

沿着centerline插入等距点（10，2）->（50，2）

删除重复点

eq_spaced_points:在0-1之间插入等距点，所有的点为50个

pxy centerline的x,y值

每个centerline之间的距离

归一化

[0，归一化的距离逐个值相加]





