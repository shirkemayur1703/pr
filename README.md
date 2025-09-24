Perfect 👌 You already have the schema (UserInfo, Product, Cart) in Oracle.
Let’s go step by step:


---

⚙️ Step 1: Configure Oracle Database Connection in Java (Maven project, not Spring Boot)

1. Add Oracle JDBC Driver to Maven

Oracle driver is not available in Maven Central (due to licensing). You have two options:

Option A: Manually install driver into Maven local repo

1. Download ojdbc8.jar (for Java 8+).


2. Run this command:

mvn install:install-file -Dfile=ojdbc8.jar -DgroupId=com.oracle.database.jdbc \
  -DartifactId=ojdbc8 -Dversion=19.8.0.0 -Dpackaging=jar


3. Add dependency in pom.xml:

<dependency>
    <groupId>com.oracle.database.jdbc</groupId>
    <artifactId>ojdbc8</artifactId>
    <version>19.8.0.0</version>
</dependency>



Option B: Add ojdbc8.jar directly to your project’s lib/ folder
and include it in classpath (works but not Maven-friendly).



---

2. Create Database Connection Utility

package com.example.shopping.util;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class DBConnection {
    private static final String URL = "jdbc:oracle:thin:@localhost:1521:xe"; // Change SID/service
    private static final String USER = "your_username";
    private static final String PASSWORD = "your_password";

    public static Connection getConnection() throws SQLException {
        try {
            Class.forName("oracle.jdbc.driver.OracleDriver");
        } catch (ClassNotFoundException e) {
            throw new RuntimeException("Oracle JDBC Driver not found!", e);
        }
        return DriverManager.getConnection(URL, USER, PASSWORD);
    }
}

✅ This will be reused across all DAOs.


---

🛒 Step 2: DAO Layer

We’ll implement DAO classes for Product and Cart with CRUD operations you requested.


---

1. ProductDAO

package com.example.shopping.dao;

import com.example.shopping.util.DBConnection;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

public class ProductDAO {

    // Add new product
    public void addProduct(String prodId, String name, String category, double price, double discount, String description) {
        String sql = "INSERT INTO Product (prod_id, prod_name, category, price, discount, description) VALUES (?, ?, ?, ?, ?, ?)";
        try (Connection conn = DBConnection.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, prodId);
            ps.setString(2, name);
            ps.setString(3, category);
            ps.setDouble(4, price);
            ps.setDouble(5, discount);
            ps.setString(6, description);
            ps.executeUpdate();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    // Search by name
    public List<String> searchByName(String keyword) {
        String sql = "SELECT prod_name FROM Product WHERE LOWER(prod_name) LIKE ?";
        List<String> results = new ArrayList<>();
        try (Connection conn = DBConnection.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, "%" + keyword.toLowerCase() + "%");
            ResultSet rs = ps.executeQuery();
            while (rs.next()) {
                results.add(rs.getString("prod_name"));
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return results;
    }

    // Search by category
    public List<String> searchByCategory(String category) {
        String sql = "SELECT prod_name FROM Product WHERE LOWER(category) = ?";
        List<String> results = new ArrayList<>();
        try (Connection conn = DBConnection.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, category.toLowerCase());
            ResultSet rs = ps.executeQuery();
            while (rs.next()) {
                results.add(rs.getString("prod_name"));
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return results;
    }

    // Search by price range
    public List<String> searchByPriceRange(double minPrice, double maxPrice) {
        String sql = "SELECT prod_name FROM Product WHERE price BETWEEN ? AND ?";
        List<String> results = new ArrayList<>();
        try (Connection conn = DBConnection.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setDouble(1, minPrice);
            ps.setDouble(2, maxPrice);
            ResultSet rs = ps.executeQuery();
            while (rs.next()) {
                results.add(rs.getString("prod_name"));
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return results;
    }
}


---

2. CartDAO

package com.example.shopping.dao;

import com.example.shopping.util.DBConnection;
import java.sql.*;

public class CartDAO {

    // Add product to cart
    public void addToCart(String userId, String productId, int quantity) {
        String sql = "MERGE INTO Cart c " +
                     "USING (SELECT ? AS user_id, ? AS product_id FROM dual) d " +
                     "ON (c.user_id = d.user_id AND c.product_id = d.product_id) " +
                     "WHEN MATCHED THEN UPDATE SET c.quantity = c.quantity + ? " +
                     "WHEN NOT MATCHED THEN INSERT (user_id, product_id, quantity) VALUES (?, ?, ?)";
        try (Connection conn = DBConnection.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, userId);
            ps.setString(2, productId);
            ps.setInt(3, quantity);
            ps.setString(4, userId);
            ps.setString(5, productId);
            ps.setInt(6, quantity);
            ps.executeUpdate();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    // Update quantity
    public void updateQuantity(String userId, String productId, int quantity) {
        String sql = "UPDATE Cart SET quantity = ? WHERE user_id = ? AND product_id = ?";
        try (Connection conn = DBConnection.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setInt(1, quantity);
            ps.setString(2, userId);
            ps.setString(3, productId);
            ps.executeUpdate();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    // Remove product from cart
    public void removeFromCart(String userId, String productId) {
        String sql = "DELETE FROM Cart WHERE user_id = ? AND product_id = ?";
        try (Connection conn = DBConnection.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, userId);
            ps.setString(2, productId);
            ps.executeUpdate();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}


---

🚀 Step 3: How it Works

Add product → ProductDAO.addProduct(...)

Search product → ProductDAO.searchByName("phone"), searchByCategory("electronics"), searchByPriceRange(1000, 5000)

Add to cart → CartDAO.addToCart(userId, productId, quantity)

Update quantity → CartDAO.updateQuantity(userId, productId, newQuantity)

Remove product → CartDAO.removeFromCart(userId, productId)



---

👉 This setup matches exactly the schema in your screenshot and handles all required operations.

Would you like me to also create a Service Layer (CartService & ProductService) on top of DAOs so that your future controller (API layer) won’t directly talk to DAOs? That will make your backend cleaner.

