https://fra.cloud.appwrite.io/v1
690684600011bfe2dca5
69068f6200047f243ccf
articles
69069119001b6f871014
        
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
