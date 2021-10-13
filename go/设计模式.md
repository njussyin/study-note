# 一、单例模式
## 概念：
保证一个类仅有一个实例，并提供一个访问它的全局访问点。
![image](https://user-images.githubusercontent.com/46216099/136938805-2afb489d-5f77-49bc-82b3-51e3acc7bf0d.png)

## 特点：
1、有GetInstance方法可以从外部获取到该实例。<br>
2、外部无法创建实例，构造函数是私有的，无法从外部访问。

## 使用场景：
1、多线程情况下会造成资源访问冲突的（例如log，需要写文件的）<br>
2、需要全局保持唯一的（例如配置信息）

## 优点：
1、避免频繁的创建和销毁实例，减少内存消耗 <br>
2、避免对资源的多重占用

## 设计考虑点：
1、创建对象时是否是线程安全的（多个线程同时调用，不会生成多个实例）<br>
2、是否支持延迟加载（？？？不确定，指的是只有在需要用的时候才去延迟生成实例，而不是提前生成放在那里可能用不到？）<br>
3、GetInstance的效率是否高

## 实现方式
1、饿汉式（在项目init的时候直接创建好）<br>
2、懒汉式
```
package design

import (
	"fmt"
	"sync"
)

type singleTon struct { // 这里一定要用小写，不然在外面就可以初始化了
}

func (s *singleTon) Show() {
	fmt.Println("hello world")
}

var (
	once   sync.Once
	single *singleTon // 这里一定要用小写，不然在外面就直接调用，拿到还没有初始化好的nil
)

func GetSingleInstance() *singleTon {
	once.Do(func() {
		single = &singleTon{}
	})
	return single
}
func main() {
	single := design.GetSingleInstance()
	single.Show()
}
```

## 项目应用实例：
1、config采用的是饿汉式
```
// 初始化
func Init() {
	hub.Lock()             // 加锁防止重复初始化
	defer hub.Unlock()
	if hub.Inited {
		return
	}
	hub.Inited = true

	var err error

	// Init config，项目启动时就初始化好了config
	hub.NeoConfig, err = config.InitNeoConfig(appYML)
	utils.Must(err)
  ......
}

// 获取
func GetNeoConfig() *config.Configer {
	hub.RLock()
	defer hub.RUnlock()
	if hub.NeoConfig == nil {
		panic(ErrNeoConfigNotLoad)
	}
	return hub.NeoConfig
}
```
2、redis连接池
```
func NewPoolManager(_logger *log.NeoLogger, redisSettings cc.Configer) (*PoolManager, error) {
	//var err error
	redisSettingsKV := redisSettings.KV()
	pm := &PoolManager{
		pools:   make(map[string]*Client, len(redisSettingsKV)),
		configs: make(map[string]cc.Configer, len(redisSettingsKV)),
		logger:  _logger,
	}
	for name, _ := range redisSettingsKV {
		redisConfig := redisSettings.Config(name)
		pm.configs[name] = redisConfig
    
    // ******* 最初服务启动的时候，每个服务对所有的Redis都创建了连接池，显然是没必要的，大部分的服务只需要连接少量的几个redis
		//if redisConfig.BoolOr("enable_log", false) {
		//	pm.pools[name], err = NewPooledClient(logger, redisConfig)
		//} else {
		//	pm.pools[name], err = NewPooledClient(nil, redisConfig)
		//}
		//if err != nil {
		//	return nil, err
		//}
	}

	return pm, nil
}

// 在需要用到某个Redis的时候通过此方法获取，并在获取不到的时候创建
// GetPooledClient returns pool by name, nil returned if not found.
func (pm *PoolManager) GetPooledClient(name string) *Client {
	pm.Lock() // 同样加锁防止重复初始化
	defer pm.Unlock()
	if pool, ok := pm.pools[name]; ok {
		return pool
	}
	redisConfig := pm.configs[name]
	if redisConfig == nil {
		panic(fmt.Sprintf("redis config name:%s not found", name))
	}
	var err error
	if redisConfig.BoolOr("enable_log", false) {
		pm.pools[name], err = NewPooledClient(pm.logger, redisConfig)
	} else {
		pm.pools[name], err = NewPooledClient(nil, redisConfig)
	}
	if err != nil {
		panic(err)
	}
	return pm.pools[name]
}
```
