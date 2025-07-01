import { Args } from '@/runtime';

// 定义输入参数接口
interface Input {
  query: string;                      // 搜索关键词
  page?: number;                      // 页码（百度API通过top_k控制分页，此处为兼容设计）
  pageSize?: number;                  // 每页结果数
  siteFilter?: string[];              // 站点过滤
  timeFilter?: 'week' | 'month' | 'semiyear' | 'year'; // 时间过滤
}

// 定义百度搜索结果项接口
interface BaiduResultItem {
  title: string;        // 标题
  url: string;          // 链接
  snippet: string;      // 摘要
  date?: string;        // 发布时间
  source?: string;      // 来源站点
}

// 定义输出参数接口
interface Output {
  results: BaiduResultItem[];  // 搜索结果列表
  message: string;             // 状态消息
  total?: number;              // 总结果数（百度未直接返回，此处为兼容字段）
  requestId?: string;          // 请求ID
}

/**
 * 百度搜索插件元数据
 * @metadata
 * {
 *   "name": "BaiduSearchV2",
 *   "description": "基于百度AI搜索V2接口执行搜索，返回搜索结果",
 *   "input": {
 *     "query": {
 *       "type": "string",
 *       "description": "要发送到百度的搜索查询词",
 *       "required": true
 *     },
 *     "page": {
 *       "type": "integer",
 *       "description": "页码（通过pageSize计算偏移量）",
 *       "optional": true,
 *       "default": 1
 *     },
 *     "pageSize": {
 *       "type": "integer",
 *       "description": "每页结果数（最大50）",
 *       "optional": true,
 *       "default": 30
 *     },
 *     "siteFilter": {
 *       "type": "array",
 *       "items": { "type": "string" },
 *       "description": "站点过滤",
 *       "optional": true
 *     },
 *     "timeFilter": {
 *       "type": "string",
 *       "description": "时间过滤（week/month/semiyear/year）",
 *       "optional": true,
 *       "enum": ["week", "month", "semiyear", "year"]
 *     }
 *   },
 *   "output": {
 *     "results": {
 *       "type": "array",
 *       "items": {
 *         "type": "object",
 *         "properties": {
 *           "title": { "type": "string", "description": "搜索结果标题" },
 *           "url": { "type": "string", "description": "结果链接" },
 *           "snippet": { "type": "string", "description": "结果摘要" },
 *           "date": { "type": "string", "description": "发布时间", "optional": true },
 *           "source": { "type": "string", "description": "来源站点", "optional": true }
 *         }
 *       },
 *       "description": "搜索结果列表"
 *     },
 *     "message": {
 *       "type": "string",
 *       "description": "指示搜索结果的状态消息"
 *     },
 *     "total": {
 *       "type": "integer",
 *       "description": "搜索结果总数",
 *       "optional": true
 *     },
 *     "requestId": {
 *       "type": "string",
 *       "description": "百度API请求ID",
 *       "optional": true
 *     }
 *   }
 * }
 */

/**
 * 百度搜索插件处理函数
 * @param {Object} args.input - 包含搜索参数的输入对象
 * @param {Object} args.logger - 日志记录器
 * @returns {Promise<Output>} 搜索结果和状态消息
 */
export async function handler({ input, logger }: Args<Input>): Promise<Output> {
  const { 
    query, 
    page = 1, 
    pageSize = 20, 
    siteFilter = [], 
    timeFilter 
  } = input;

  // 输入验证
  if (!query) {
    logger?.error?.('未提供搜索查询词');
    return {
      results: [],
      message: '错误：搜索查询词是必需的',
    };
  }

  const adjustedPageSize = Math.min(Math.floor(pageSize), 50);
  if (pageSize > 50) {
    logger?.warn?.('pageSize超过最大值50，自动调整为50');
  }

  try {
    logger?.info?.(`执行百度AI搜索，查询词：${query}，页码：${page}`);
    
    // 构建请求参数
    const requestBody = {
      messages: [
        {
          role: 'user',
          content: query
        }
      ],
      search_source: 'baidu_search_v2',
      resource_type_filter: [{ type: 'web', top_k: adjustedPageSize }],
      // 可选参数配置
      ...(timeFilter && {
        search_recency_filter: timeFilter
      })
    };

    // 百度API认证信息（建议从环境变量获取）
    const API_KEY = process.env.BAIDU_API_KEY || '在这里输入百度的API';
    const API_URL = 'https://qianfan.baidubce.com/v2/ai_search/chat/completions';

    // 发送API请求
    const response = await fetch(API_URL, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(requestBody)
    });

    if (!response.ok) {
      const errorText = await response.text();
      throw new Error(`百度API调用失败，状态码：${response.status}，错误信息：${errorText}`);
    }

    const data = await response.json();
    logger?.debug?.('百度API原始响应数据：', data);

    // 解析响应结果
    const results = (data.references || [])
      .map((item: any) => {
        try {
          // 提取域名（处理各种URL格式）
          const url = item.url || '#';
          const domain = new URL(url).hostname;
          
          return {
            title: item.title || '无标题',
            url,
            snippet: item.content || '无摘要',
            date: item.date,
            source: item.web_anchor || domain
          };
        } catch (e) {
          // URL解析失败时，使用默认值
          return {
            title: item.title || '无标题',
            url: item.url || '#',
            snippet: item.content || '无摘要',
            date: item.date,
            source: item.web_anchor || '未知来源'
          };
        }
      });

    // 计算结果总数
    const resultCount = results.length;
    
    return {
      results,
      message: `成功获取${resultCount}条搜索结果，查询词：${query}`,
      total: resultCount,
      requestId: data.requestId || undefined
    };
  } catch (error: any) {
    logger?.error?.(`百度搜索过程中出错：${error.message}`);
    return {
      results: [],
      message: `错误：搜索失败。${error.message}`,
      requestId: undefined
    };
  }
}
