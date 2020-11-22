## REST API

GET -> 查（Read)

Post -> 增（Create）

Put -> 改(update)

Delete -> 删(Delete)

## API设计

### 用户

1. **创建用户**: 
    1. URL:/user 
    2. *Method*:Post ,
    3.  SC: 201(created), 400(bad request), 500(inside error)
2. **用户登录**:
    1.  URL:/user/:username 
    2. *Method*: Post, 
    3. SC: 200, 400
3. **获取用户基本信息**: 
    1. URL:/user/:username 
    2. *Methcd*: GET; 
    3. SC: 200, 401, 400, 403, 500 
4. **用户注销**: 
    1. URL:/user/:username 
    2. Method: DELETE, 
    3. SC: 204, 400, 401, 403, 500



### 用户资源

*   List all videos: URL: /user/:username/videos
    *   Method: GET 
    *   SC: 200, 400, 500
*   Get One Video:  URL: /user/:username/videos/:vid-id
    *   Method: GET
    *   SC: 200, 400, 500
*   Delete One Video:  URL: /user/:username/videos/:vid-id
    *   Method: DELETE
    *   SC: 204, 400, 401, 403, 500



### 用户评论

*   Show comments:
    *   URL: /videos/:vid-id/comments
    *   Method: Get
    *   Sc: 200, 400, 500
*   Post a comment:
    *   URL: /videos/:vid-id/comments
    *   Method: Post
    *   SC: 201, 400, 500
*   Delete a comment:
    *   URL: /videos/:vid-id/comments
    *   Method: Delete
    *   SC: 204, 400, 401, 403, 500