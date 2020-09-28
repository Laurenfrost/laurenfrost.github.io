---
title: Barefoot 中的隐马尔可夫地图匹配详解
date: 2020-09-24 17:21:00
tags: 隐马尔科夫模型
---

### 基础

隐马尔可夫模型（Hidden Markov Model，缩写成 HMM）假设系统的状态只能通过时间来间接测量。

在地图匹配（map matching）中，我们有一组位置测量值（position measurement），比如说 GPS 数据。这组测量值其实就隐含了这个被测物体在地图上移动轨迹的信息，比如说它经过了哪条路、拐了哪几个弯。这个过程其实是符合上述的隐马尔可夫模型的假设的。所以用它来解决地图匹配问题也变得理所当然。

隐马尔可夫地图匹配（HMM Map Matching）是一种非常鲁棒的方法，可以很准确地从位置测量中猜测出被测物体在地图上移动轨迹。[<sup>1</sup>](#refer-1) [<sup>2</sup>](#refer-2) 当然，系统已知地图上道路的位置和它们之间的连接。

有一组位置测量 _z<sub>0</sub>_, _z<sub>1</sub>_, ..., _z<sub>T</sub>_，其中时间 _t_ _(0 &le; t &le; T)_ 下的一个测量就是 _z<sub>t</sub>_

A sequence of position measurements _z<sub>0</sub>_, _z<sub>1</sub>_, ..., _z<sub>T</sub>_ is map matched by finding for each measurement _z<sub>t</sub>_, made at some time _t_ _(0 &le; t &le; T)_, its most likely matching on the map _s<sub>t</sub><sup>i</sup>_ from a set of ___matching candidates___ _S<sub>t</sub> = {s<sub>t</sub><sup>1</sup>, ..., s<sub>t</sub><sup>n</sup>}_. A set of matching candidates _S<sub>t</sub>_ is here referred to as a ___candidate vector___. For each consecutive pair of candidate vectors _S<sub>t</sub>_ and _S<sub>t+1</sub>_ _(0 &le; t &lt; T)_, there is a transition between each pair of matching candidates _s<sub>t</sub><sup>i</sup>_ and _s<sub>t+1</sub><sup>j</sup>_ which is the route between map positions in the map. An illustration of the HMM with matching candidates and transitions is shown in the figure below.

Barefoot's map matching API consists of four main components. This includes a matcher component that performs map matching with a HMM filter iteratively for each position measurement _z<sub>t</sub>_ of an object. It also includes a state memory component that stores candidate vectors _S<sub>t</sub>_ and their probabilities _p_; and it can be accessed to get the current position estimate _s&#773;<sub>t</sub>_ or the most likely path (_s<sub>0</sub>_ ... _s<sub>t</sub>_). Further, it includes a map component for spatial search of matching candidates _S<sub>t</sub>_ near the measured position _z<sub>t</sub>_; and a router component to find routes _&lang;s<sub>t-1</sub>,s<sub>t</sub>&rang;_ between pairs of candidates _(s<sub>t-1</sub>,s<sub>t</sub>)_.

<p align="center">
<img src="https://github.com/bmwcarit/barefoot/raw/master/doc-files/com/bmwcarit/barefoot/matcher/matcher-components.png?raw=true" width="600">
</p>

地图匹配的一个迭代周期内，程序只需要处理给定的位置测量值（position measurement）_z<sub>t</sub>_，然后更新状态即可。这个过程具体包含下述四步操作：

1. 给定一个位置测量值 _z<sub>t</sub>_ 在地图中对**候选匹配**（matching candidate）_s<sub>t</sub> &#8712; S<sub>t</sub>_ 进行空间搜索，_S<sub>t</sub>_ 为时间 _t_ 时的**候选向量**（candidate vector）。

2. 从内存中抓取 _S<sub>t-1</sub>_，如果没有就返回空向量。

3. 对每一组候选匹配 _(s<sub>t-1</sub>, s<sub>t</sub>)_，找到路径 route _&lang;s<sub>t-1</sub>,s<sub>t</sub>&rang;_，该路径即为候选匹配间的**转换**（transition）。其中 _s<sub>t-1</sub> &#8712; S<sub>t-1</sub>_ and _s<sub>t</sub> &#8712; S<sub>t</sub>_。

4. 对候选匹配 _s<sub>t</sub>_ 计算**过滤器概率**（filter probability）和**序列概率**（sequence probability），并更新内存中的概率 _p_ 和候选向量 _S<sub>t</sub>_。

下面是 map matching API 的一个小例子：

```Java
import com.bmwcarit.barefoot.roadmap.Loader;
import com.bmwcarit.barefoot.roadmap.Road;
import com.bmwcarit.barefoot.roadmap.RoadMap;
import com.bmwcarit.barefoot.roadmap.RoadPoint;
import com.bmwcarit.barefoot.roadmap.TimePriority;
import com.bmwcarit.barefoot.spatial.Geography;
import com.bmwcarit.barefoot.topology.Dijkstra;

// Load and construct road map
// 从数据库中加载并构造地图
// 配置文件里包含访问数据库所须的账号及密码
RoadMap map = Loader.roadmap("config/oberbayern.properties", true).construct();

// Instantiate matcher and state data structure
// 实例化 matcher 和 状态数据
Matcher matcher = new Matcher(map, new Dijkstra<Road, RoadPoint>(), new TimePriority(), new Geography());

// Input as sample batch (offline) or sample stream (online)
// 批量输入sample（离线），或者，输入sample流（在线）
// 这里的在线可能指的是机器学习中“在线学习”的“在线”。
List<MatcherSample> samples = readSamples();

// Match full sequence of samples
// 匹配sample中的全序列
MatcherKState state = matcher.mmatch(samples, 1, 500);

// Access map matching result: sequence for all samples
// 访问地图匹配的结果：所有sample的序列
for (MatcherCandidate cand : state.sequence()) {
    cand.point().edge().base().refid(); // OSM id
    cand.point().edge().base().id(); // road id
    cand.point().edge().heading(); // heading
    cand.point().geometry(); // GPS position (on the road)
    if (cand.transition() != null)
        cand.transition().route().geometry(); // path geometry from last matching candidate
}
```

在线地图匹配则需要每个迭代周期更新一次状态数据：

```Java
// Create initial (empty) state memory
// 分配一段空间来存储状态数据
MatcherKState state = new MatcherKState();

// Iterate over sequence (stream) of samples
// 按sample顺序迭代
for (MatcherSample sample : samples) {
	// Execute matcher with single sample and update state memory
    state.update(matcher.execute(state.vector(), state.sample(), sample), sample);

    // Access map matching result: estimate for most recent sample
    MatcherCandidate estimate = state.estimate();
    System.out.println(estimate.point().edge().base().refid()); // OSM id
} 
```

### k-State 数据结构

k-State 数据结构用于存储状态数据，它包含**候选向量**（即一组**候选匹配**），并提供了如下数据的访问：

+ **匹配估计**（estimate），即在时间 _t_ 时最有可能的那个候选匹配 _s&#773;<sub>t</sub>_，它代表了目标当前位置在地图上的估计。
+ **序列估计**（estimate of sequence），即最有可能的候选匹配序列，它代表了目标在地图上最有可能的移动路径。

初始状态下 k-State 数据为空，需要状态向量 _S<sub>t</sub>_ 来更新它。一开始是 _S<sub>0</sub>_，它包含一组候选匹配（图中用圆圈表示）和一个指向匹配估计的指针（图中用加粗圆圈表示）。

下一个迭代周期，matcher 抓取 _S<sub>0</sub>_。matcher 决定每个候选匹配 _s<sub>1</sub><sub>i</sub> &#8712; S<sub>1</sub>_ 的过滤器概率和序列概率，以及它们各自的前任候选匹配 _s<sub>0</sub><sub>i</sub> &#8712; S<sub>0</sub>_。

接下来，用这个新的状态向量 _S<sub>1</sub>_ 来更新 k-State 状态数据，顺带更新指向匹配估计的指针和最有可能的序列（粗箭头）。

<p align="center">
<img src="https://github.com/bmwcarit/barefoot/raw/master/doc-files/com/bmwcarit/barefoot/markov/kstate-1.png?raw=true" width="150" hspace="40">
<img src="https://github.com/bmwcarit/barefoot/raw/master/doc-files/com/bmwcarit/barefoot/markov/kstate-2.png?raw=true" width="150" hspace="40">
<img src="https://github.com/bmwcarit/barefoot/raw/master/doc-files/com/bmwcarit/barefoot/markov/kstate-3.png?raw=true" width="150" hspace="40">
</p>

后续的迭代过程基本都是重复上述过程：
* matcher 抓取 _S<sub>t-1</sub>_。
* 计算每个候选匹配 _s<sub>t</sub><sub>i</sub> &#8712; S<sub>t</sub>_ 的过滤器概率和序列概率。
* 计算每个候选匹配最有可能的各自的前任候选匹配 _s<sub>0</sub><sub>i</sub> &#8712; S<sub>0</sub>_。
* 更新 k-State 状态数据，删除上个迭代中的多余数据（没在序列上的的候选匹配）。

<p align="center">
<img src="https://github.com/bmwcarit/barefoot/raw/master/doc-files/com/bmwcarit/barefoot/markov/kstate-4.png?raw=true" width="150" hspace="40">
<img src="https://github.com/bmwcarit/barefoot/raw/master/doc-files/com/bmwcarit/barefoot/markov/kstate-5.png?raw=true" width="150" hspace="40">
<img src="https://github.com/bmwcarit/barefoot/raw/master/doc-files/com/bmwcarit/barefoot/markov/kstate-6.png?raw=true" width="150" hspace="40">
</p>

### json 格式

k-State 的 json 表达由参数 `k` `t` 以及两个 json 数组 `sequence` 和 `candidates`。

+ `sequence` 包含每一次采样的测量信息和候选向量。
  + `sample` 就是某一次采样的位置测量信息。
  + `vector` 对应该次测量的候选向量。
    + `candid` 是候选向量中，某个 id。
    + `predid` 则是这个候选匹配的前一个候选匹配的 id。

+ `candidates` 包含候选匹配的信息，以及过滤器概率和序列概率。
  + `candidate` 的 `id` 就是 `candid` 和 `predid` 所引用的 id。
  + `count` 仅用于k-State数据结构的收敛，是指后继候选向量中以该匹配候选向量作为其前身的匹配候选向量的数目。
  + `point` 为地图中的位置。
  + `road` 则是道路的 id。
  + `frac` 是 fraction。
  + `transition` 如果存在，则它表示从上一匹配到当前匹配的路径 `route`。
    + `source` 表示路径的起点，`target` 表示路径的终点。
      + `road` 和 `frac` 进一步记录 `source` 和 `target` 在地图上的准确位置。
    + `roads` 则具体记录该路径所经过的道路 id。

举个例子：

```json
{
  "sequence": [
    {
      "sample": {
        "id": "a1396ab7-7caa-4c31-9f3c-8982055e3de6",
        "point": "POINT (11.536577179945997 48.14905556426255)",
        "time": 1410325357000
      },
      "vector": [
        {
          "candid": "e649f976-564a-4760-9a74-c82ba6c4653e",
          "predid": ""
        }
      ]
    },
    {
      "sample": {
        "id": "a1396ab7-7caa-4c31-9f3c-8982055e3de6",
        "point": "POINT (11.536219651738836 48.14672536176703)",
        "time": 1410325372000
      },
      "vector": [
        {
          "candid": "648cd380-f317-4ebb-b9e2-650a80054bf7",
          "predid": "e649f976-564a-4760-9a74-c82ba6c4653e"
        },
        {
          "candid": "6b347e77-eb92-43d3-a60d-69d9bb71f9d4",
          "predid": "e649f976-564a-4760-9a74-c82ba6c4653e"
        }
      ]
    }
  ],
  "candidates": [
    {
      "count": 2,
      "candidate": {
        "filtprob": 0.11565717758307356,
        "id": "e649f976-564a-4760-9a74-c82ba6c4653e",
        "point": {
          "frac": 0.4104557158596576,
          "road": 9362030
        },
        "seqprob": -1.0999901830140701
      }
    },
    {
      "count": 0,
      "candidate": {
        "filtprob": 0.2370833183857761,
        "id": "648cd380-f317-4ebb-b9e2-650a80054bf7",
        "point": {
          "frac": 0.06531311234979269,
          "road": 8533290
        },
        "seqprob": -3.2870414276380666,
        "transition": {
          "route": {
            "roads": [
              9362030,
              ...
              8533290
            ],
            "source": {
              "frac": 0.4104557158596576,
              "road": 9362030
            },
            "target": {
              "frac": 0.06531311234979269,
              "road": 8533290
            }
          }
        }
      }
    },
    ...
  ],
  "k": -1,
  "t": -1
}
```




### 参考文献

<div id="refer-1"></div>
<p>[1] P. Newson and J. Krumm. Hidden Markov Map Matching Through Noise and Sparseness. In Proceedings of International Conference on Advances in Geographic Information Systems, 2009.</p>

<div id="refer-2"></div>
<p>[2] C.Y. Goh, J. Dauwels, N. Mitrovic, M.T. Asif, A. Oran, and P. Jaillet. Online map-matching based on Hidden Markov model for real-time traffic sensing applications. In International IEEE Conference on Intelligent Transportation Systems, 2012.</p>