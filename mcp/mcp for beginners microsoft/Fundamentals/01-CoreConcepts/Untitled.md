# MCP Tool Design Exercise: E-commerce Product Performance Analyzer

## 1. Tool Name

**`analyzeProductPerformance`**

## 2. Parameters

```typescript
{
  productId: z.string().describe("Unique product identifier"),
  timeRange: z.enum(['7d', '30d', '90d', '1y']).default('30d').describe("Analysis time period"),
  metrics: z.array(z.enum(['sales', 'revenue', 'views', 'conversion', 'reviews', 'inventory']))
    .default(['sales', 'revenue', 'conversion'])
    .describe("Metrics to analyze"),
  compareWith: z.string().optional().describe("Another product ID for comparison"),
  segmentBy: z.enum(['category', 'region', 'customer_type', 'none']).default('none')
    .describe("How to segment the analysis"),
  includeRecommendations: z.boolean().default(true)
    .describe("Whether to include AI-powered recommendations")
}
```

## 3. Expected Output

```json
{
  "product": {
    "id": "PROD-12345",
    "name": "Wireless Bluetooth Headphones",
    "category": "Electronics > Audio",
    "status": "active"
  },
  "timeRange": "30d",
  "performance": {
    "sales": {
      "total_units": 1250,
      "trend": "+15.3%",
      "daily_average": 41.7,
      "peak_day": "2024-07-10"
    },
    "revenue": {
      "total": 87500.00,
      "trend": "+18.2%",
      "average_order_value": 70.00,
      "currency": "USD"
    },
    "conversion": {
      "rate": 3.2,
      "trend": "+0.8%",
      "total_views": 39062,
      "total_purchases": 1250
    }
  },
  "comparison": {
    "product_id": "PROD-67890",
    "product_name": "Wired Gaming Headset",
    "performance_vs_comparison": {
      "sales": "+23.5%",
      "revenue": "+31.2%",
      "conversion": "-0.4%"
    }
  },
  "segmentation": {
    "by_region": [
      {
        "region": "North America",
        "sales": 625,
        "revenue": 43750.00,
        "percentage": 50.0
      },
      {
        "region": "Europe", 
        "sales": 375,
        "revenue": 26250.00,
        "percentage": 30.0
      },
      {
        "region": "Asia Pacific",
        "sales": 250,
        "revenue": 17500.00,
        "percentage": 20.0
      }
    ]
  },
  "insights": {
    "strengths": [
      "Strong sales growth (+15.3%) outperforming category average (+8.1%)",
      "High customer satisfaction with 4.6/5 average rating",
      "Excellent performance in North American market"
    ],
    "concerns": [
      "Conversion rate slightly below category average (3.2% vs 3.5%)",
      "Inventory levels getting low (estimated 2 weeks remaining)",
      "Declining performance in Europe region (-5% vs last period)"
    ]
  },
  "recommendations": [
    {
      "type": "inventory",
      "priority": "high",
      "action": "Restock inventory within 1 week to avoid stockouts",
      "impact": "Prevent potential $25K revenue loss"
    },
    {
      "type": "marketing", 
      "priority": "medium",
      "action": "Launch targeted campaign in Europe to boost regional sales",
      "impact": "Potential +10% sales increase in underperforming region"
    },
    {
      "type": "pricing",
      "priority": "low", 
      "action": "Consider slight price increase (+5%) given strong demand",
      "impact": "Potential +3.5% revenue increase with minimal sales impact"
    }
  ],
  "metadata": {
    "generated_at": "2024-07-17T10:30:00Z",
    "data_freshness": "2024-07-17T09:00:00Z",
    "confidence_score": 0.92
  }
}
```

## 4. How AI Models Would Use This Tool

### Scenario 1: Performance Review Meeting

**User:** "How is our top-selling headphones product doing this month?"

**AI Model Process:**

1. Calls `analyzeProductPerformance` with productId for headphones, timeRange='30d'
2. Receives comprehensive performance data
3. Generates executive summary highlighting key metrics and trends
4. Presents actionable insights and recommendations

**AI Response:** "Your Wireless Bluetooth Headphones (PROD-12345) are performing excellently this month with 1,250 units sold (+15.3% growth) and $87,500 revenue (+18.2%). However, I notice two areas needing attention: inventory is running low (2 weeks remaining) and Europe sales are declining. I recommend immediate restocking and a targeted European marketing campaign."

### Scenario 2: Competitive Analysis

**User:** "Compare our headphones with the gaming headset - which should we promote more?"

**AI Model Process:**

1. Calls `analyzeProductPerformance` with compareWith parameter
2. Analyzes relative performance metrics
3. Evaluates market positioning and profitability
4. Provides strategic recommendation

**AI Response:** "Your Wireless Bluetooth Headphones significantly outperform the Gaming Headset with +23.5% more sales and +31.2% higher revenue, though with slightly lower conversion rates. Given the stronger momentum and broader market appeal, I recommend prioritizing the Bluetooth Headphones for your next promotion campaign."

### Scenario 3: Strategic Planning

**User:** "What products need immediate attention for Q3 planning?"

**AI Model Process:**

1. Calls `analyzeProductPerformance` for multiple key products
2. Identifies patterns, risks, and opportunities across portfolio
3. Prioritizes actions based on business impact
4. Creates strategic roadmap

**AI Response:** "Based on the analysis, three products require immediate Q3 attention: 1) Bluetooth Headphones need urgent restocking (high priority), 2) Gaming Mouse shows declining trends requiring marketing intervention (medium), 3) Smartphone Cases have optimization opportunities with pricing adjustments (low). Here's your prioritized action plan..."

## Key Benefits

✅ **Comprehensive Analytics**: Single tool provides 360° view of product performance ✅ **Actionable Insights**: AI-powered recommendations with business impact estimates  
✅ **Flexible Segmentation**: Analyze by different dimensions for deeper insights ✅ **Competitive Intelligence**: Built-in comparison capabilities ✅ **Real-time Decision Support**: Fresh data with confidence scoring ✅ **Scalable**: Works for individual products or portfolio analysis