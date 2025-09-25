Perfect 👍 let’s extend your DAO + Service with the missing cart operations.

We’ll keep it clean and simple, with Oracle in mind.


---

🔹 DAO Layer

CartItemDao

import java.sql.SQLException;
import java.util.List;

public interface CartItemDao {
    CartItem findByUserAndProduct(String userId, String productId) throws SQLException;
    void insert(CartItem cartItem) throws SQLException;
    void update(CartItem cartItem) throws SQLException;
    void delete(String userId, String productId) throws SQLException;
    List<CartItem> findByUser(String userId) throws SQLException;
    void clearCart(String userId) throws SQLException;
}

CartItemDaoImpl

import java.sql.*;
import java.util.*;

public class CartItemDaoImpl implements CartItemDao {
    private final Connection connection;

    public CartItemDaoImpl(Connection connection) {
        this.connection = connection;
    }

    @Override
    public CartItem findByUserAndProduct(String userId, String productId) throws SQLException {
        String sql = "SELECT quantity FROM cart WHERE user_id = ? AND product_id = ?";
        try (PreparedStatement ps = connection.prepareStatement(sql)) {
            ps.setString(1, userId);
            ps.setString(2, productId);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) {
                return new CartItem(userId, productId, rs.getInt("quantity"));
            }
        }
        return null;
    }

    @Override
    public void insert(CartItem cartItem) throws SQLException {
        String sql = "INSERT INTO cart(user_id, product_id, quantity) VALUES (?, ?, ?)";
        try (PreparedStatement ps = connection.prepareStatement(sql)) {
            ps.setString(1, cartItem.getUserId());
            ps.setString(2, cartItem.getProductId());
            ps.setInt(3, cartItem.getQuantity());
            ps.executeUpdate();
        }
    }

    @Override
    public void update(CartItem cartItem) throws SQLException {
        String sql = "UPDATE cart SET quantity = ? WHERE user_id = ? AND product_id = ?";
        try (PreparedStatement ps = connection.prepareStatement(sql)) {
            ps.setInt(1, cartItem.getQuantity());
            ps.setString(2, cartItem.getUserId());
            ps.setString(3, cartItem.getProductId());
            ps.executeUpdate();
        }
    }

    @Override
    public void delete(String userId, String productId) throws SQLException {
        String sql = "DELETE FROM cart WHERE user_id = ? AND product_id = ?";
        try (PreparedStatement ps = connection.prepareStatement(sql)) {
            ps.setString(1, userId);
            ps.setString(2, productId);
            ps.executeUpdate();
        }
    }

    @Override
    public List<CartItem> findByUser(String userId) throws SQLException {
        List<CartItem> items = new ArrayList<>();
        String sql = "SELECT c.product_id, c.quantity, p.name, p.price " +
                     "FROM cart c JOIN product p ON c.product_id = p.prodid " +
                     "WHERE c.user_id = ?";
        try (PreparedStatement ps = connection.prepareStatement(sql)) {
            ps.setString(1, userId);
            ResultSet rs = ps.executeQuery();
            while (rs.next()) {
                CartItem item = new CartItem(
                    userId,
                    rs.getString("product_id"),
                    rs.getInt("quantity")
                );
                // Optional: set product details if CartItem holds a Product
                items.add(item);
            }
        }
        return items;
    }

    @Override
    public void clearCart(String userId) throws SQLException {
        String sql = "DELETE FROM cart WHERE user_id = ?";
        try (PreparedStatement ps = connection.prepareStatement(sql)) {
            ps.setString(1, userId);
            ps.executeUpdate();
        }
    }
}


---

🔹 Service Layer

CartService

import java.sql.SQLException;
import java.util.List;

public class CartService {
    private final CartItemDao cartItemDao;
    private final ProductDao productDao; // assume you have a DAO for products

    public CartService(CartItemDao cartItemDao, ProductDao productDao) {
        this.cartItemDao = cartItemDao;
        this.productDao = productDao;
    }

    // Add or increase quantity
    public void addToCart(String userId, String productId, int qty) throws SQLException {
        CartItem existing = cartItemDao.findByUserAndProduct(userId, productId);
        if (existing != null) {
            existing.setQuantity(existing.getQuantity() + qty);
            cartItemDao.update(existing);
        } else {
            cartItemDao.insert(new CartItem(userId, productId, qty));
        }
    }

    // Decrease quantity
    public void decreaseQuantity(String userId, String productId) throws SQLException {
        CartItem existing = cartItemDao.findByUserAndProduct(userId, productId);
        if (existing != null) {
            int newQty = existing.getQuantity() - 1;
            if (newQty > 0) {
                existing.setQuantity(newQty);
                cartItemDao.update(existing);
            } else {
                cartItemDao.delete(userId, productId);
            }
        }
    }

    // Remove item entirely
    public void removeItem(String userId, String productId) throws SQLException {
        cartItemDao.delete(userId, productId);
    }

    // View cart
    public List<CartItem> viewCart(String userId) throws SQLException {
        return cartItemDao.findByUser(userId);
    }

    // Calculate total price
    public double calculateTotal(String userId) throws SQLException {
        List<CartItem> cartItems = cartItemDao.findByUser(userId);
        double total = 0.0;
        for (CartItem item : cartItems) {
            Product product = productDao.findById(item.getProductId());
            total += product.getPrice() * item.getQuantity();
        }
        return total;
    }

    // Clear entire cart
    public void clearCart(String userId) throws SQLException {
        cartItemDao.clearCart(userId);
    }
}


---

🔹 Summary of Supported Features

Add item / increase quantity

Decrease quantity (remove row if zero)

Remove item directly

View cart (returns list of CartItem)

Calculate total (cart subtotal)

Clear cart



---

👉 With this, your cart is functionally complete for a typical e-commerce project.

Do you want me to also extend this with stock check logic (so a user can’t add more items than available in inventory)?

