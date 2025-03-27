```go
package web_api

import (
	"github.com/gin-gonic/gin"
	"gorm.io/gorm"
	"strconv"
)

var db *gorm.DB

func GetList() {
	r := gin.Default() //配置默认路由引擎

	r.GET("/ListOpLog", ListOpLog) //定义列表接口路由，用ListOpLog函数来实现
	r.GET("/ListOpLog/Detail", DetailOpLog)

	r.Run(":8080") //运行程序
}

// 列表接口
func ListOpLog(c *gin.Context) {
	//1.获取请求参数
	Page := c.DefaultQuery("page", "1")                      //页码，默认为1
	PageSize := c.DefaultQuery("page_size", "10")            //页面大小，默认为10
	OrderBy := c.DefaultQuery("order_by", "created_at DESC") //排序字段，默认为按时间降序排序，否则返回传入从参数值

	OpStatus := c.Query("op_status")          //操作状态，作为筛选条件，此外还有，，IP地址，系统模块，操作类型，请求方式
	OpLogId := c.Query("op_log_id")           //日志编号
	OpPeopelName := c.Query("op_people_name") //操作人员
	IpAddress := c.Query("ip_address")        //IP地址
	ModuleName := c.Query("module_name")      //系统模块
	OpType := c.Query("op_type")              //操作类型
	RequestModel := c.Query("request_model")  //请求方式

	//2.查询数据库
	var logs []SysOpLog            //用来存储查询到的日志记录
	query := db.Model(&SysOpLog{}) //创建一个查询对象
	if OpStatus != "" {
		query = query.Where("op_status = ?", OpStatus) //如果OpStatus不为空，添加筛选条件
	}
	if OpPeopelName != "" {
		query = query.Where("op_people_name = ?", OpPeopelName)
	}
	if OpLogId != "" {
		query = query.Where("op_log_id = ?", OpLogId)
	}
	if IpAddress != "" {
		query = query.Where("ip_address = ?", IpAddress)
	}
	if ModuleName != "" {
		query = query.Where("module_name = ?", ModuleName)
	}
	if OpType != "" {
		query = query.Where("op_type = ?", OpType)
	}
	if RequestModel != "" {
		query = query.Where("request_model = ?", RequestModel)
	}

	//3.执行分页和排序
	PageInt, _ := strconv.Atoi(Page) //数据类型转换
	PageSizeInt, _ := strconv.Atoi(PageSize)

	offset := (PageInt - 1) * PageSizeInt                          //计算在第几页，前面需要跳过多少数据
	query = query.Offset(offset).Limit(PageSizeInt).Order(OrderBy) //在第几页，每页多少条，以及排序方式
	//4.执行查询
	var total int64
	query.Count(&total)
	//将结果存在logs切片中，如果失败则返回状态码和错误信息
	if err := query.Find(&logs).Error; err != nil {
		c.JSON(500, gin.H{"ERROR:\t数据库查询失败\n": err})
		return
	}
	//5.返回结果
	//返回状态码和JSON格式的响应
	c.JSON(200, gin.H{
		"total": len(logs), //日志总数
		"data":  logs,      //查询到的日志列表
	})

}

// 详情接口查看
func DetailOpLog(c *gin.Context) {
	//1.获取请求参数,日志Id
	id := c.Param("op_log_id")
	if id == "" {
		c.JSON(400, gin.H{"Error:日志ID不能为空": "\n"})
	}
	//2.查询数据库
	var log SysOpLog
	if err := db.First(&log, id).Error; err != nil { //查询单条记录，&用来储存查到的日志
		c.JSON(404, gin.H{"Error:日志不存在\n": err}) //如果没有找到，则返回404
		return
	}
	//3.返回结果
	c.JSON(200, log)
}

type SysOpLog struct { //日志表结构：日志表id，创建时间，更新时间，删除时间，操作者id，操作者ip，操作时间，操作类型，操作状态,
	gorm.Model            //包含创建者ID，创建时间，更新时间和删除时间
	OpLogId        int    `gorm:"type:int;not null" mapstructure:"op_log_id"`                //日志表id
	OpType         string `gorm:"type:varchar(128);not null" mapstructure:"op_type"`         //操作类型
	ModuleName     string `gorm:"type:varchar(128);not null" mapstructure:"module_name"`     //系统模块
	RequestModel   string `gorm:"type:varchar(128);not null" mapstructure:"request"`         //请求方式
	OpStatus       string `gorm:"type:varchar(128);not null" mapstructure:"op_status"`       //操作状态
	OpMethod       string `gorm:"type:varchar(256);not null "mapstructure:"op_method"`       //操作方法
	RequestAddress string `gorm:"type:varchar(128);not null" mapstructure:"request_address"` //请求地址
	RequestData    string `gorm:"type:varchar(128);not null" mapstructure:"request_data"`    //请求参数
	ReturnAnswer   string `gorm:"type:varchar(128);not null" mapstructure:"return_answer"`   //返回结果
	IpAddress      string `gorm:"type:varchar(128);not null" mapstructure:"ip_address"`      //IP地址
	OpPeopleName   string `gorm:"type:varchar(128);not null" mapstructure:"op_people_name"`  //操作人用户名称
	OpTime         string `gorm:"column:op_time;not null"mapstructure:"op_time"`             //操作日期
}

func (SysOpLog) TableName() string {
	return "s_op_log"
}

```

