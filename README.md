# social-media-api
# social-media-api
# User Entity
public class User
{
    public int Id { get; set; }
    public string Username { get; set; }
    public string Email { get; set; }
    public string PasswordHash { get; set; }

    public ICollection<Post> Posts { get; set; }
    public ICollection<Follow> Followers { get; set; }
    public ICollection<Follow> Following { get; set; }
}

# Post Entity
public class Post
{
    public int Id { get; set; }
    public string Content { get; set; }
    public DateTime CreatedAt { get; set; }

    public int UserId { get; set; }
    public User User { get; set; }

    public ICollection<Comment> Comments { get; set; }
    public ICollection<Like> Likes { get; set; }
}

# DTO
public class CreatePostDto
{
    public string Content { get; set; }
}

# JWT
"Jwt": {
  "Key": "THIS_IS_A_SUPER_SECRET_KEY_CHANGE_IT",
  "Issuer": "SocialHubAPI",
  "Audience": "SocialHubClient",
  "ExpiryMinutes": 60
}

public class JwtSettings
{
    public string Key { get; set; } = string.Empty;
    public string Issuer { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public int ExpiryMinutes { get; set; }
}

using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var jwtSettings = builder.Configuration
    .GetSection("Jwt")
    .Get<JwtSettings>();

builder.Services.AddSingleton(jwtSettings!);

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,

            ValidIssuer = jwtSettings!.Issuer,
            ValidAudience = jwtSettings.Audience,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwtSettings.Key)
            ),

            ClockSkew = TimeSpan.Zero
        };
    });

    app.UseAuthentication();
app.UseAuthorization();

using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using Microsoft.IdentityModel.Tokens;
using System.Text;

public interface ITokenService
{
    string GenerateToken(Guid userId, string email);
}

public class TokenService : ITokenService
{
    private readonly JwtSettings _jwt;

    public TokenService(JwtSettings jwt)
    {
        _jwt = jwt;
    }

    public string GenerateToken(Guid userId, string email)
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, userId.ToString()),
            new Claim(ClaimTypes.Email, email)
        };

        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_jwt.Key)
        );

        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _jwt.Issuer,
            audience: _jwt.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_jwt.ExpiryMinutes),
            signingCredentials: creds
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
builder.Services.AddScoped<ITokenService, TokenService>();
[Authorize]
[ApiController]
[Route("api/posts")]
public class PostsController : ControllerBase
{
}

# Follow Entity
public class Follow
{
    public int FollowerId { get; set; }   // Who follows
    public User Follower { get; set; }

    public int FollowingId { get; set; }  // Who is being followed
    public User Following { get; set; }

    public DateTime CreatedAt { get; set; }
}

# User Change Entity
public class User
{
    public int Id { get; set; }
    public string Username { get; set; }

    public ICollection<Follow> Followers { get; set; }
    public ICollection<Follow> Following { get; set; }
}

# EF Core relationships
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Follow>()
        .HasKey(f => new { f.FollowerId, f.FollowingId });

    modelBuilder.Entity<Follow>()
        .HasOne(f => f.Follower)
        .WithMany(u => u.Following)
        .HasForeignKey(f => f.FollowerId)
        .OnDelete(DeleteBehavior.Restrict);

    modelBuilder.Entity<Follow>()
        .HasOne(f => f.Following)
        .WithMany(u => u.Followers)
        .HasForeignKey(f => f.FollowingId)
        .OnDelete(DeleteBehavior.Restrict);
}

# Following

public class FollowService : IFollowService
{
    private readonly IFollowRepository _followRepo;

    public FollowService(IFollowRepository followRepo)
    {
        _followRepo = followRepo;
    }

    public async Task FollowAsync(int followerId, int followingId)
    {
        if (followerId == followingId)
            throw new Exception("You cannot follow yourself.");

        if (await _followRepo.ExistsAsync(followerId, followingId))
            return;

        await _followRepo.AddAsync(new Follow
        {
            FollowerId = followerId,
            FollowingId = followingId,
            CreatedAt = DateTime.UtcNow
        });
    }

    public async Task UnfollowAsync(int followerId, int followingId)
    {
        await _followRepo.RemoveAsync(followerId, followingId);
    }
}
[ApiController]
[Route("api/follows")]
[Authorize]
public class FollowsController : ControllerBase
{
    private readonly IFollowService _followService;

    public FollowsController(IFollowService followService)
    {
        _followService = followService;
    }

    [HttpPost("{userId}")]
    public async Task<IActionResult> FollowUser(int userId)
    {
        var currentUserId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier));
        await _followService.FollowAsync(currentUserId, userId);
        return Ok();
    }

    [HttpDelete("{userId}")]
    public async Task<IActionResult> UnfollowUser(int userId)
    {
        var currentUserId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier));
        await _followService.UnfollowAsync(currentUserId, userId);
        return NoContent();
    }
}

# Comment Service
public class Comment
{
    public int Id { get; set; }
    public string Content { get; set; }
    public DateTime CreatedAt { get; set; }

    public int UserId { get; set; }
    public User User { get; set; }

    public int PostId { get; set; }
    public Post Post { get; set; }
}
}
public class CreateCommentDto
{
    public string Content { get; set; }
}
public interface ICommentRepository
{
    Task AddAsync(Comment comment);
    Task<IEnumerable<Comment>> GetByPostIdAsync(int postId);
}
public class CommentRepository : ICommentRepository
{
    private readonly AppDbContext _context;

    public CommentRepository(AppDbContext context)
    {
        _context = context;
    }
    public class CommentService : ICommentService
{
    private readonly ICommentRepository _repo;

    public CommentService(ICommentRepository repo)
    {
        _repo = repo;
    }

    public async Task AddCommentAsync(int userId, int postId, string content)
    {
        var comment = new Comment
        {
            UserId = userId,




# Like Service
public class Like
{
    public int UserId { get; set; }
    public User User { get; set; }

    public int PostId { get; set; }
    public Post Post { get; set; }

    public DateTime CreatedAt { get; set; }
}
}
public interface LikeRepository
{
    Task<bool> ExistsAsync(int userId, int postId);
    Task AddAsync(Like like);
    Task RemoveAsync(int userId, int postId);
}

# Post Service
public class Post
{
    public int Id { get; set; }
    public string Content { get; set; }

    public ICollection<Comment> Comments { get; set; }
    public ICollection<Like> Likes { get; set; }
}
public class LikeService : ILikeService
{
    private readonly ILikeRepository _repo;

    public LikeService(ILikeRepository repo)
    {
        _repo = repo;
    }

    public async Task LikeAsync(int userId, int postId)
    {
        if (await _repo.ExistsAsync(userId, postId))
            return;

        await _repo.AddAsync(new Like
        {
            UserId = userId,
            PostId = postId,
            CreatedAt = DateTime.UtcNow
        });
    }

# Controllers
[ApiController]
[Route("api/posts/{postId}/comments")]
[Authorize]
public class CommentsController : ControllerBase
{
    private readonly ICommentService _service;

    public CommentsController(ICommentService service)
    {
        _service = service;
    }

    [HttpPost]
    public async Task<IActionResult> AddComment(int postId, CreateCommentDto dto)
    {
        var userId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier));
        await _service.AddCommentAsync(userId, postId, dto.Content);
        return Ok();
    }
}
[ApiController]
[Route("api/posts/{postId}/likes")]
[Authorize]
public class LikesController : ControllerBase
{
    private readonly ILikeService _service;

    public LikesController(ILikeService service)
    {
        _service = service;
    }

    [HttpPost]
    public async Task<IActionResult> LikePost(int postId)
    {
        var userId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier));
        await _service.LikeAsync(userId, postId);
        return Ok();
    }

    [HttpDelete]
    public async Task<IActionResult> UnlikePost(int postId)
    {
        var userId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier));
        await _service.UnlikeAsync(userId, postId);
        return NoContent();
    }
}

# User Search
public interface IUserRepository
{
    Task<IEnumerable<User>> SearchAsync(string query);
}
public async Task<IEnumerable<User>> SearchAsync(string query)
{
    return await _context.Users
        .Where(u => u.Username.Contains(query))
        .Take(20)
        .ToListAsync();
}

# Post Search
public interface IPostRepository
{
    Task<IEnumerable<Post>> SearchAsync(string query);
}
public async Task<IEnumerable<Post>> SearchAsync(string query)
{
    return await _context.Posts
        .Where(p => p.Content.Contains(query))
        .OrderByDescending(p => p.CreatedAt)
        .Take(20)
        .ToListAsync();
}

# Search Service
public class SearchService : ISearchService
{
    private readonly IUserRepository _userRepo;
    private readonly IPostRepository _postRepo;

    public SearchService(IUserRepository userRepo, IPostRepository postRepo)
    {
        _userRepo = userRepo;
        _postRepo = postRepo;
    }

    public async Task<SearchResultDto> SearchAsync(string query)
    {
        return new SearchResultDto
        {
            Users = await _userRepo.SearchAsync(query),
            Posts = await _postRepo.SearchAsync(query)
        };
    }
}

# Search Controller
[ApiController]
[Route("api/search")]
public class SearchController : ControllerBase
{
    private readonly ISearchService _service;

    public SearchController(ISearchService service)
    {
        _service = service;
    }

    [HttpGet("users")]
    public async Task<IActionResult> SearchUsers([FromQuery] string q)
    {
        return Ok(await _service.SearchUsersAsync(q));
    }

    [HttpGet("posts")]
    public async Task<IActionResult> SearchPosts([FromQuery] string q)
    {
        return Ok(await _service.SearchPostsAsync(q));
    }
}

# Data Models
public class Follow
{
    public Guid FollowerId { get; set; }
    public Guid FollowingId { get; set; }
}

public class Post
{
    public Guid Id { get; set; }
    public Guid UserId { get; set; }
    public string Content { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }

    public ICollection<Like> Likes { get; set; } = new List<Like>();
    public ICollection<Comment> Comments { get; set; } = new List<Comment>();
}

# DTO
public class FeedPostDto
{
    public Guid PostId { get; set; }
    public Guid UserId { get; set; }
    public string Content { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }

    public int LikesCount { get; set; }
    public int CommentsCount { get; set; }
}

# Service Interface
public interface IFeedService
{
    Task<IReadOnlyList<FeedPostDto>> GetUserFeedAsync(
        Guid userId,
        int page,
        int pageSize);
}

# Service Implementation
public class FeedService : IFeedService
{
    private readonly AppDbContext _context;

    public FeedService(AppDbContext context)
    {
        _context = context;
    }

    public async Task<IReadOnlyList<FeedPostDto>> GetUserFeedAsync(
        Guid userId,
        int page,
        int pageSize)
    {
        var followedUserIds = _context.Follows
            .Where(f => f.FollowerId == userId)
            .Select(f => f.FollowingId);

        return await _context.Posts
            .Where(p => followedUserIds.Contains(p.UserId))
            .OrderByDescending(p => p.CreatedAt)
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .Select(p => new FeedPostDto
            {
                PostId = p.Id,
                UserId = p.UserId,
                Content = p.Content,
                CreatedAt = p.CreatedAt,
                LikesCount = p.Likes.Count,
                CommentsCount = p.Comments.Count
            })
            .AsNoTracking()
            .ToListAsync();
    }
}

# Controller Endpoint
[ApiController]
[Route("api/feed")]
[Authorize]
public class FeedController : ControllerBase
{
    private readonly IFeedService _feedService;

    public FeedController(IFeedService feedService)
    {
        _feedService = feedService;
    }

    [HttpGet]
    public async Task<IActionResult> GetFeed(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 10)
    {
        var userId = Guid.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier)!);

        var feed = await _feedService.GetUserFeedAsync(userId, page, pageSize);
        return Ok(feed);
    }
}

# Dependency Injection
builder.Services.AddScoped<IFeedService, FeedService>();

# Unit Testing
public static class TestDbFactory
{
    public static AppDbContext Create()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString())
            .Options;

        return new AppDbContext(options);
    }
}
public class FeedServiceTests
{
    [Fact]
    public async Task GetUserFeedAsync_Returns_Posts_From_Followed_Users()
    {
        // Arrange
        var context = TestDbFactory.Create();
        var service = new FeedService(context);

        var user = Guid.NewGuid();
        var followedUser = Guid.NewGuid();

        context.Follows.Add(new Follow
        {
            FollowerId = user,
            FollowingId = followedUser
        });

        context.Posts.Add(new Post
        {
            Id = Guid.NewGuid(),
            UserId = followedUser,
            Content = "Hello world",
            CreatedAt = DateTime.UtcNow
        });

        await context.SaveChangesAsync();

        // Act
        var result = await service.GetUserFeedAsync(user, 1, 10);

        // Assert
        Assert.Single(result);
        Assert.Equal("Hello world", result[0].Content);
    }
}
[Fact]
public async Task Feed_Is_Ordered_By_Newest_First()
{
    var context = TestDbFactory.Create();
    var service = new FeedService(context);

    var user = Guid.NewGuid();
    var followed = Guid.NewGuid();

    context.Follows.Add(new Follow { FollowerId = user, FollowingId = followed });

    context.Posts.AddRange(
        new Post { UserId = followed, Content = "Old", CreatedAt = DateTime.UtcNow.AddDays(-1) },
        new Post { UserId = followed, Content = "New", CreatedAt = DateTime.UtcNow }
    );

    await context.SaveChangesAsync();

    var feed = await service.GetUserFeedAsync(user, 1, 10);

    Assert.Equal("New", feed[0].Content);
}
# Pagination Index
CREATE INDEX IX_Posts_UserId_CreatedAt
ON Posts (UserId, CreatedAt DESC);



