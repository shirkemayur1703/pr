// Generic DAO Interface - for Product, User, etc.
public interface DAOInterface<T> {
    void create(T entity);
    T findById(Long id);
    List<T> findAll();
    void update(T entity);  // Makes sense for Product, User
    void delete(Long id);
}

// CartItem-specific interface - business operations
public interface CartItemDAO {
    void addCartItem(CartItem cartItem);           // Add/merge logic
    void increaseQty(Long userId, Long prodId, Double price);
    void decreaseQty(Long userId, Long prodId, Double price);
    CartItem getCartItem(Long userId, Long prodId);
    List<CartItem> getCartItemsByUserId(Long userId);
    void removeFromCart(Long userId, Long prodId);
    // NO generic update() method!
}

// CartItem Model
public class CartItem {
    private Long userId;
    private Long prodId;
    private Integer qty;
    private Double amount;
    
    // Constructors
    public CartItem() {}
    
    public CartItem(Long userId, Long prodId, Integer qty, Double amount) {
        this.userId = userId;
        this.prodId = prodId;
        this.qty = qty;
        this.amount = amount;
    }
    
    // Getters and Setters
    public Long getUserId() {
        return userId;
    }
    
    public void setUserId(Long userId) {
        this.userId = userId;
    }
    
    public Long getProdId() {
        return prodId;
    }
    
    public void setProdId(Long prodId) {
        this.prodId = prodId;
    }
    
    public Integer getQty() {
        return qty;
    }
    
    public void setQty(Integer qty) {
        this.qty = qty;
    }
    
    public Double getAmount() {
        return amount;
    }
    
    public void setAmount(Double amount) {
        this.amount = amount;
    }
    
    @Override
    public String toString() {
        return "CartItem{" +
                "userId=" + userId +
                ", prodId=" + prodId +
                ", qty=" + qty +
                ", amount=" + amount +
                '}';
    }
}

// Product Model
public class Product {
    private Long prodId;
    private Double price;
    private Double discount;
    private String name;
    private String description;
    private String category;
    
    // Constructors
    public Product() {}
    
    public Product(Long prodId, Double price, Double discount, String name, String description, String category) {
        this.prodId = prodId;
        this.price = price;
        this.discount = discount;
        this.name = name;
        this.description = description;
        this.category = category;
    }
    
    // Getters and Setters
    public Long getProdId() {
        return prodId;
    }
    
    public void setProdId(Long prodId) {
        this.prodId = prodId;
    }
    
    public Double getPrice() {
        return price;
    }
    
    public void setPrice(Double price) {
        this.price = price;
    }
    
    public Double getDiscount() {
        return discount;
    }
    
    public void setDiscount(Double discount) {
        this.discount = discount;
    }
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
    
    public String getDescription() {
        return description;
    }
    
    public void setDescription(String description) {
        this.description = description;
    }
    
    public String getCategory() {
        return category;
    }
    
    public void setCategory(String category) {
        this.category = category;
    }
}

// CartItem DAO Interface
import java.util.List;

public interface CartItemDAO {
    void addCartItem(CartItem cartItem) throws Exception;
    void increaseQty(Long userId, Long prodId, Double discountedPrice) throws Exception;
    void decreaseQty(Long userId, Long prodId, Double discountedPrice) throws Exception;
    void updateAmount(Long userId, Long prodId, Double amount) throws Exception;
    List<CartItem> getCartItemsByUserId(Long userId) throws Exception;
    List<CartItem> getAllCartItems() throws Exception;
    void deleteCartItem(Long userId, Long prodId) throws Exception;
    CartItem getCartItem(Long userId, Long prodId) throws Exception;
}

// CartItem DAO Implementation
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

public class CartItemDAOImpl implements CartItemDAO {
    private Connection connection;
    
    public CartItemDAOImpl(Connection connection) {
        this.connection = connection;
    }
    
    @Override
    public void addCartItem(CartItem cartItem) throws Exception {
        String sql = """
            MERGE INTO CartItem c
            USING (SELECT ? as user_id, ? as prod_id, ? as qty, ? as amount FROM dual) src
            ON (c.user_id = src.user_id AND c.prod_id = src.prod_id)
            WHEN MATCHED THEN
                UPDATE SET qty = c.qty + src.qty, amount = src.amount
            WHEN NOT MATCHED THEN
                INSERT (user_id, prod_id, qty, amount)
                VALUES (src.user_id, src.prod_id, src.qty, src.amount)
            """;
            
        try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
            pstmt.setLong(1, cartItem.getUserId());
            pstmt.setLong(2, cartItem.getProdId());
            pstmt.setInt(3, cartItem.getQty());
            pstmt.setDouble(4, cartItem.getAmount());
            pstmt.executeUpdate();
        }
    }
    
    @Override
    public void increaseQty(Long userId, Long prodId, Double discountedPrice) throws Exception {
        String sql = "UPDATE CartItem SET qty = qty + 1, amount = (qty + 1) * ? WHERE user_id = ? AND prod_id = ?";
        
        try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
            pstmt.setDouble(1, discountedPrice);
            pstmt.setLong(2, userId);
            pstmt.setLong(3, prodId);
            int rowsUpdated = pstmt.executeUpdate();
            
            if (rowsUpdated == 0) {
                throw new Exception("Cart item not found for user: " + userId + ", product: " + prodId);
            }
        }
    }
    
    @Override
    public void decreaseQty(Long userId, Long prodId, Double discountedPrice) throws Exception {
        // First check current quantity
        CartItem item = getCartItem(userId, prodId);
        if (item == null) {
            throw new Exception("Cart item not found");
        }
        
        if (item.getQty() <= 1) {
            // Delete item if quantity would become 0
            deleteCartItem(userId, prodId);
        } else {
            // Decrease quantity and update amount in single query
            String sql = "UPDATE CartItem SET qty = qty - 1, amount = (qty - 1) * ? WHERE user_id = ? AND prod_id = ?";
            try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
                pstmt.setDouble(1, discountedPrice);
                pstmt.setLong(2, userId);
                pstmt.setLong(3, prodId);
                pstmt.executeUpdate();
            }
        }
    }
    
    @Override
    public void updateAmount(Long userId, Long prodId, Double amount) throws Exception {
        String sql = "UPDATE CartItem SET amount = ? WHERE user_id = ? AND prod_id = ?";
            
        try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
            pstmt.setDouble(1, amount);
            pstmt.setLong(2, userId);
            pstmt.setLong(3, prodId);
            int rowsUpdated = pstmt.executeUpdate();
            
            if (rowsUpdated == 0) {
                throw new Exception("Cart item not found for amount update");
            }
        }
    }
    
    @Override
    public CartItem getCartItem(Long userId, Long prodId) throws Exception {
        String sql = "SELECT user_id, prod_id, qty, amount FROM CartItem WHERE user_id = ? AND prod_id = ?";
        
        try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
            pstmt.setLong(1, userId);
            pstmt.setLong(2, prodId);
            ResultSet rs = pstmt.executeQuery();
            
            if (rs.next()) {
                CartItem item = new CartItem();
                item.setUserId(rs.getLong("user_id"));
                item.setProdId(rs.getLong("prod_id"));
                item.setQty(rs.getInt("qty"));
                item.setAmount(rs.getDouble("amount"));
                return item;
            }
        }
        
        return null;
    }
    
    @Override
    public List<CartItem> getCartItemsByUserId(Long userId) throws Exception {
        String sql = "SELECT user_id, prod_id, qty, amount FROM CartItem WHERE user_id = ? ORDER BY prod_id";
        List<CartItem> cartItems = new ArrayList<>();
        
        try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
            pstmt.setLong(1, userId);
            ResultSet rs = pstmt.executeQuery();
            
            while (rs.next()) {
                CartItem item = new CartItem();
                item.setUserId(rs.getLong("user_id"));
                item.setProdId(rs.getLong("prod_id"));
                item.setQty(rs.getInt("qty"));
                item.setAmount(rs.getDouble("amount"));
                cartItems.add(item);
            }
        }
        
        return cartItems;
    }
    
    @Override
    public List<CartItem> getAllCartItems() throws Exception {
        String sql = "SELECT user_id, prod_id, qty, amount FROM CartItem ORDER BY user_id, prod_id";
        List<CartItem> cartItems = new ArrayList<>();
        
        try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
            ResultSet rs = pstmt.executeQuery();
            
            while (rs.next()) {
                CartItem item = new CartItem();
                item.setUserId(rs.getLong("user_id"));
                item.setProdId(rs.getLong("prod_id"));
                item.setQty(rs.getInt("qty"));
                item.setAmount(rs.getDouble("amount"));
                cartItems.add(item);
            }
        }
        
        return cartItems;
    }
    
    @Override
    public void deleteCartItem(Long userId, Long prodId) throws Exception {
        String sql = "DELETE FROM CartItem WHERE user_id = ? AND prod_id = ?";
        
        try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
            pstmt.setLong(1, userId);
            pstmt.setLong(2, prodId);
            int rowsDeleted = pstmt.executeUpdate();
            
            if (rowsDeleted == 0) {
                throw new Exception("Cart item not found for deletion");
            }
        }
    }
}

// Product DAO Interface
import java.util.List;

public interface ProductDAO {
    Product getProductById(Long prodId) throws Exception;
    List<Product> getAllProducts() throws Exception;
    List<Product> getProductsByCategory(String category) throws Exception;
    void addProduct(Product product) throws Exception;
    void updateProduct(Product product) throws Exception;
    void deleteProduct(Long prodId) throws Exception;
    boolean productExists(Long prodId) throws Exception;
}

// Product DAO Implementation
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

public class ProductDAOImpl implements ProductDAO {
    private Connection connection;
    
    public ProductDAOImpl(Connection connection) {
        this.connection = connection;
    }
    
    @Override
    public Product getProductById(Long prodId) throws Exception {
        String sql = "SELECT prod_id, price, discount, name, description, category FROM Product WHERE prod_id = ?";
        
        try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
            pstmt.setLong(1, prodId);
            ResultSet rs = pstmt.executeQuery();
            
            if (rs.next()) {
                Product product = new Product();
                product.setProdId(rs.getLong("prod_id"));
                product.setPrice(rs.getDouble("price"));
                product.setDiscount(rs.getDouble("discount"));
                product.setName(rs.getString("name"));
                product.setDescription(rs.getString("description"));
                product.setCategory(rs.getString("category"));
                return product;
            }
        }
        
        return null;
    }
    
    @Override
    public boolean productExists(Long prodId) throws Exception {
        String sql = "SELECT 1 FROM Product WHERE prod_id = ?";
        
        try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
            pstmt.setLong(1, prodId);
            ResultSet rs = pstmt.executeQuery();
            return rs.next();
        }
    }
    
    @Override
    public List<Product> getAllProducts() throws Exception {
        String sql = "SELECT prod_id, price, discount, name, description, category FROM Product ORDER BY prod_id";
        List<Product> products = new ArrayList<>();
        
        try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
            ResultSet rs = pstmt.executeQuery();
            
            while (rs.next()) {
                Product product = new Product();
                product.setProdId(rs.getLong("prod_id"));
                product.setPrice(rs.getDouble("price"));
                product.setDiscount(rs.getDouble("discount"));
                product.setName(rs.getString("name"));
                product.setDescription(rs.getString("description"));
                product.setCategory(rs.getString("category"));
                products.add(product);
            }
        }
        
        return products;
    }
    
    @Override
    public List<Product> getProductsByCategory(String category) throws Exception {
        String sql = "SELECT prod_id, price, discount, name, description, category FROM Product WHERE category = ? ORDER BY name";
        List<Product> products = new ArrayList<>();
        
        try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
            pstmt.setString(1, category);
            ResultSet rs = pstmt.executeQuery();
            
            while (rs.next()) {
                Product product = new Product();
                product.setProdId(rs.getLong("prod_id"));
                product.setPrice(rs.getDouble("price"));
                product.setDiscount(rs.getDouble("discount"));
                product.setName(rs.getString("name"));
                product.setDescription(rs.getString("description"));
                product.setCategory(rs.getString("category"));
                products.add(product);
            }
        }
        
        return products;
    }
    
    @Override
    public void addProduct(Product product) throws Exception {
        String sql = "INSERT INTO Product (prod_id, price, discount, name, description, category) VALUES (?, ?, ?, ?, ?, ?)";
        
        try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
            pstmt.setLong(1, product.getProdId());
            pstmt.setDouble(2, product.getPrice());
            
            if (product.getDiscount() != null) {
                pstmt.setDouble(3, product.getDiscount());
            } else {
                pstmt.setNull(3, Types.DOUBLE);
            }
            
            pstmt.setString(4, product.getName());
            pstmt.setString(5, product.getDescription());
            pstmt.setString(6, product.getCategory());
            pstmt.executeUpdate();
        }
    }
    
    @Override
    public void updateProduct(Product product) throws Exception {
        String sql = "UPDATE Product SET price = ?, discount = ?, name = ?, description = ?, category = ? WHERE prod_id = ?";
        
        try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
            pstmt.setDouble(1, product.getPrice());
            
            if (product.getDiscount() != null) {
                pstmt.setDouble(2, product.getDiscount());
            } else {
                pstmt.setNull(2, Types.DOUBLE);
            }
            
            pstmt.setString(3, product.getName());
            pstmt.setString(4, product.getDescription());
            pstmt.setString(5, product.getCategory());
            pstmt.setLong(6, product.getProdId());
            
            int rowsUpdated = pstmt.executeUpdate();
            if (rowsUpdated == 0) {
                throw new Exception("Product not found for update: " + product.getProdId());
            }
        }
    }
    
    @Override
    public void deleteProduct(Long prodId) throws Exception {
        String sql = "DELETE FROM Product WHERE prod_id = ?";
        
        try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
            pstmt.setLong(1, prodId);
            int rowsDeleted = pstmt.executeUpdate();
            
            if (rowsDeleted == 0) {
                throw new Exception("Product not found for deletion: " + prodId);
            }
        }
    }
}

// CartItem Service - Business Logic Layer
import java.util.List;

public class CartItemService {
    private CartItemDAO cartItemDAO;
    private ProductDAO productDAO;
    
    public CartItemService(CartItemDAO cartItemDAO, ProductDAO productDAO) {
        this.cartItemDAO = cartItemDAO;
        this.productDAO = productDAO;
    }
    
    public void addToCart(Long userId, Long prodId, Integer qty) throws Exception {
        // Validate product exists and get product details
        Product product = productDAO.getProductById(prodId);
        if (product == null) {
            throw new Exception("Product not found: " + prodId);
        }
        
        // Calculate amount with discount
        Double discountedPrice = calculateDiscountedPrice(product);
        Double amount = discountedPrice * qty;
        
        CartItem cartItem = new CartItem(userId, prodId, qty, amount);
        cartItemDAO.addCartItem(cartItem);
        
        // If item was merged (already existed), recalculate amount for new total quantity
        CartItem updatedItem = cartItemDAO.getCartItem(userId, prodId);
        if (updatedItem != null) {
            Double newAmount = discountedPrice * updatedItem.getQty();
            cartItemDAO.updateAmount(userId, prodId, newAmount);
        }
    }
    
    public void increaseCartItemQty(Long userId, Long prodId) throws Exception {
        // Get product for price calculation
        Product product = productDAO.getProductById(prodId);
        if (product == null) {
            throw new Exception("Product not found: " + prodId);
        }
        
        Double discountedPrice = calculateDiscountedPrice(product);
        
        // Single atomic operation - updates qty and amount together
        cartItemDAO.increaseQty(userId, prodId, discountedPrice);
    }
    
    public void decreaseCartItemQty(Long userId, Long prodId) throws Exception {
        // Get product for price calculation
        Product product = productDAO.getProductById(prodId);
        if (product == null) {
            throw new Exception("Product not found: " + prodId);
        }
        
        Double discountedPrice = calculateDiscountedPrice(product);
        
        // Single atomic operation - updates qty and amount together
        cartItemDAO.decreaseQty(userId, prodId, discountedPrice);
    }
    
    public List<CartItem> getUserCart(Long userId) throws Exception {
        return cartItemDAO.getCartItemsByUserId(userId);
    }
    
    public double calculateCartTotal(Long userId) throws Exception {
        List<CartItem> cartItems = cartItemDAO.getCartItemsByUserId(userId);
        return cartItems.stream()
                       .mapToDouble(CartItem::getAmount)
                       .sum();
    }
    
    public int getCartItemCount(Long userId) throws Exception {
        List<CartItem> cartItems = cartItemDAO.getCartItemsByUserId(userId);
        return cartItems.stream()
                       .mapToInt(CartItem::getQty)
                       .sum();
    }
    
    public void clearUserCart(Long userId) throws Exception {
        List<CartItem> cartItems = cartItemDAO.getCartItemsByUserId(userId);
        for (CartItem item : cartItems) {
            cartItemDAO.deleteCartItem(userId, item.getProdId());
        }
    }
    
    public void removeFromCart(Long userId, Long prodId) throws Exception {
        cartItemDAO.deleteCartItem(userId, prodId);
    }
    
    public void updateCartItemQty(Long userId, Long prodId, Integer newQty) throws Exception {
        if (newQty <= 0) {
            removeFromCart(userId, prodId);
        } else {
            // Get current item and calculate new amount
            Product product = productDAO.getProductById(prodId);
            if (product == null) {
                throw new Exception("Product not found: " + prodId);
            }
            
            Double discountedPrice = calculateDiscountedPrice(product);
            Double newAmount = discountedPrice * newQty;
            
            // Update quantity and amount
            CartItem currentItem = cartItemDAO.getCartItem(userId, prodId);
            if (currentItem != null) {
                // Update both quantity and amount
                CartItem updatedItem = new CartItem(userId, prodId, newQty, newAmount);
                cartItemDAO.addCartItem(updatedItem); // Will merge and update
            } else {
                throw new Exception("Cart item not found");
            }
        }
    }
    
    public CartItem getCartItem(Long userId, Long prodId) throws Exception {
        return cartItemDAO.getCartItem(userId, prodId);
    }
    
    // Admin functionality
    public List<CartItem> getAllCartItems() throws Exception {
        return cartItemDAO.getAllCartItems();
    }
    
    // Private helper method
    private Double calculateDiscountedPrice(Product product) {
        Double discount = product.getDiscount() != null ? product.getDiscount() : 0.0;
        return product.getPrice() - discount;
    }
}
