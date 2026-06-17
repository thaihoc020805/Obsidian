# 10 Ví dụ MCP Server Features

## 📑 Resources (4 ví dụ)

### 1. Contextual Data - User Profile Context

```text
context://user-profile/john-doe
```

Cung cấp thông tin về người dùng John Doe bao gồm sở thích, lịch sử mua hàng, và preferences để AI có thể cá nhân hóa phản hồi.

### 2. Knowledge Base - Technical Documentation

```text
knowledge://api-documentation/stripe-payments
```

Truy cập vào knowledge base chứa toàn bộ tài liệu API của Stripe Payments, bao gồm endpoints, parameters, và examples.

### 3. Local Files - Log Analysis

```text
file://system-logs/error-2024-07-17.log
```

Truy cập file log lỗi hệ thống để phân tích và tìm ra nguyên nhân gây sự cố.

### 4. APIs and Web Services - Weather Data

```text
api://openweathermap/current-weather/hanoi
```

Kết nối với OpenWeatherMap API để lấy thông tin thời tiết hiện tại của Hà Nội.

## 🤖 Prompts (3 ví dụ)

### 5. Templated Messages - Product Description Generator

```markdown
Tạo mô tả sản phẩm cho {{product_name}} với các đặc điểm chính:
- Loại sản phẩm: {{category}}
- Giá: {{price}}
- Tính năng nổi bật: {{features}}
- Đối tượng khách hàng: {{target_audience}}

Mô tả cần có độ dài {{word_count}} từ và tông giọng {{tone}}.
```

### 6. Pre-defined Interaction Pattern - Customer Support Flow

```markdown
Quy trình hỗ trợ khách hàng:
1. Chào hỏi: "Xin chào {{customer_name}}, tôi có thể giúp gì cho bạn?"
2. Xác định vấn đề: "Bạn đang gặp vấn đề gì với {{product_type}}?"
3. Thu thập thông tin: "Bạn có thể cung cấp {{required_info}} không?"
4. Đưa ra giải pháp: "Dựa trên thông tin bạn cung cấp, tôi đề xuất {{solution}}"
5. Xác nhận: "Giải pháp này có phù hợp với bạn không?"
```

### 7. Specialized Conversation Template - Code Review

```markdown
Đánh giá code review cho {{file_name}}:

**Phân tích:**
- Ngôn ngữ: {{language}}
- Chức năng: {{functionality}}
- Độ phức tạp: {{complexity_level}}

**Tiêu chí đánh giá:**
- Code quality: {{quality_score}}/10
- Performance: {{performance_score}}/10
- Security: {{security_score}}/10
- Maintainability: {{maintainability_score}}/10

**Đề xuất cải thiện:**
{{improvement_suggestions}}
```

## ⛏️ Tools (3 ví dụ)

### 8. Web Search Tool

```typescript
server.tool(
  "WebSearch",
  {
    query: z.string().describe("Search query"),
    limit: z.number().optional().describe("Number of results to return"),
    language: z.string().optional().describe("Language for results")
  }, 
  async (args) => {
    // Thực hiện tìm kiếm web
    const results = await searchEngine.search(args.query, {
      limit: args.limit || 10,
      language: args.language || 'vi'
    });
    
    return {
      results: results.map(item => ({
        title: item.title,
        url: item.url,
        snippet: item.description
      }))
    };
  }
)
```

### 9. Database Query Tool

```typescript
server.tool(
  "DatabaseQuery",
  {
    table: z.string().describe("Table name to query"),
    columns: z.array(z.string()).optional().describe("Columns to select"),
    where: z.string().optional().describe("WHERE condition"),
    limit: z.number().optional().describe("Limit number of results")
  },
  async (args) => {
    // Thực hiện truy vấn database
    const query = buildSQLQuery({
      table: args.table,
      columns: args.columns || ['*'],
      where: args.where,
      limit: args.limit || 100
    });
    
    const results = await database.execute(query);
    
    return {
      query: query,
      rowCount: results.length,
      data: results
    };
  }
)
```

### 10. File Operations Tool

```typescript
server.tool(
  "FileOperations",
  {
    operation: z.enum(['read', 'write', 'delete', 'list']).describe("File operation to perform"),
    path: z.string().describe("File or directory path"),
    content: z.string().optional().describe("Content to write (for write operation)"),
    encoding: z.string().optional().describe("File encoding")
  },
  async (args) => {
    // Thực hiện các thao tác file
    switch (args.operation) {
      case 'read':
        const content = await fs.readFile(args.path, args.encoding || 'utf8');
        return { content, size: content.length };
        
      case 'write':
        await fs.writeFile(args.path, args.content, args.encoding || 'utf8');
        return { success: true, message: `File written to ${args.path}` };
        
      case 'delete':
        await fs.unlink(args.path);
        return { success: true, message: `File deleted: ${args.path}` };
        
      case 'list':
        const files = await fs.readdir(args.path);
        return { files, count: files.length };
        
      default:
        throw new Error(`Unsupported operation: ${args.operation}`);
    }
  }
)
```

## Tổng kết

Các ví dụ trên minh họa 3 tính năng chính của MCP Server:

- **Resources (4 ví dụ)**: Cung cấp dữ liệu và context từ nhiều nguồn khác nhau
- **Prompts (3 ví dụ)**: Template và quy trình tương tác có cấu trúc
- **Tools (3 ví dụ)**: Các function thực thi để thực hiện các tác vụ cụ thể

Mỗi feature đều có vai trò quan trọng trong việc tăng cường khả năng tương tác giữa client, host và language model trong hệ sinh thái MCP.