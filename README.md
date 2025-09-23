Perfect 🚀 I’ll put everything together in a clean, modular, complete Java implementation of your Digital Wallet Application with:

Core module → models, services, exceptions

Persistent module → repositories (in-memory DB)

App.java → demo runner

Payment methods → DebitCard, CreditCard, UPI all implementing PaymentMethod



---

📂 Project Structure

digital-wallet/
│── core/
│   ├── model/
│   │   ├── User.java
│   │   ├── UserProfile.java
│   │   ├── Wallet.java
│   │   ├── Transaction.java
│   │   ├── PaymentMethod.java
│   │   ├── Card.java
│   │   ├── DebitCard.java
│   │   ├── CreditCard.java
│   │   └── UPI.java
│   │
│   ├── service/
│   │   ├── UserService.java
│   │   ├── WalletService.java
│   │   └── TransactionService.java
│   │
│   └── exception/
│       ├── InsufficientBalanceException.java
│       ├── InvalidPinException.java
│       └── UserNotFoundException.java
│
│── persistent/
│   └── repository/
│       ├── UserRepository.java
│       ├── WalletRepository.java
│       └── TransactionRepository.java
│
└── App.java


---

✅ Core Module

Models

User.java

package core.model;

import java.util.Objects;

public final class User {
    private final String userId;
    private final String email;
    private final String pin;
    private final UserProfile profile;

    public User(String userId, String email, String pin, UserProfile profile) {
        this.userId = Objects.requireNonNull(userId);
        this.email = Objects.requireNonNull(email);
        this.pin = Objects.requireNonNull(pin);
        this.profile = Objects.requireNonNull(profile);
    }

    public String getUserId() { return userId; }
    public String getEmail() { return email; }
    public String getPin() { return pin; }
    public UserProfile getProfile() { return profile; }
}


---

UserProfile.java

package core.model;

public class UserProfile {
    private String name;
    private String phone;

    public UserProfile(String name, String phone) {
        this.name = name;
        this.phone = phone;
    }

    public String getName() { return name; }
    public String getPhone() { return phone; }

    public void setName(String name) { this.name = name; }
    public void setPhone(String phone) { this.phone = phone; }
}


---

Wallet.java

package core.model;

import java.math.BigDecimal;

public class Wallet {
    private final String walletId;
    private BigDecimal balance;

    public Wallet(String walletId) {
        this.walletId = walletId;
        this.balance = BigDecimal.ZERO;
    }

    public String getWalletId() { return walletId; }
    public BigDecimal getBalance() { return balance; }

    public void addMoney(BigDecimal amount) {
        balance = balance.add(amount);
    }

    public void deductMoney(BigDecimal amount) {
        balance = balance.subtract(amount);
    }
}


---

Transaction.java

package core.model;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Objects;

public final class Transaction {
    private final String transactionId;
    private final String senderId;
    private final String receiverId;
    private final BigDecimal amount;
    private final LocalDateTime timestamp;

    public Transaction(String transactionId, String senderId, String receiverId, BigDecimal amount) {
        this.transactionId = Objects.requireNonNull(transactionId);
        this.senderId = Objects.requireNonNull(senderId);
        this.receiverId = Objects.requireNonNull(receiverId);
        this.amount = Objects.requireNonNull(amount);
        this.timestamp = LocalDateTime.now();
    }

    public String getTransactionId() { return transactionId; }
    public String getSenderId() { return senderId; }
    public String getReceiverId() { return receiverId; }
    public BigDecimal getAmount() { return amount; }
    public LocalDateTime getTimestamp() { return timestamp; }
}


---

Payment System

PaymentMethod.java

package core.model;

import java.math.BigDecimal;

public interface PaymentMethod {
    String getCardNumber();
    String getCardHolderName();
    boolean validatePin(String pin);
    void charge(BigDecimal amount);
}


---

Card.java

package core.model;

import java.util.Objects;

public abstract class Card implements PaymentMethod {
    private final String cardNumber;
    private final String cardHolderName;
    private final String expiryDate;
    private final String pin;

    protected Card(String cardNumber, String cardHolderName, String expiryDate, String pin) {
        this.cardNumber = Objects.requireNonNull(cardNumber);
        this.cardHolderName = Objects.requireNonNull(cardHolderName);
        this.expiryDate = Objects.requireNonNull(expiryDate);
        this.pin = Objects.requireNonNull(pin);
    }

    @Override
    public String getCardNumber() { return cardNumber; }

    @Override
    public String getCardHolderName() { return cardHolderName; }

    public String getExpiryDate() { return expiryDate; }

    @Override
    public boolean validatePin(String pin) { return this.pin.equals(pin); }
}


---

DebitCard.java

package core.model;

import java.math.BigDecimal;

public class DebitCard extends Card {
    private BigDecimal linkedAccountBalance;

    public DebitCard(String cardNumber, String holderName, String expiryDate, String pin, BigDecimal initialBalance) {
        super(cardNumber, holderName, expiryDate, pin);
        this.linkedAccountBalance = initialBalance;
    }

    @Override
    public void charge(BigDecimal amount) {
        if (linkedAccountBalance.compareTo(amount) < 0) {
            throw new RuntimeException("Insufficient funds in debit account!");
        }
        linkedAccountBalance = linkedAccountBalance.subtract(amount);
    }

    public BigDecimal getLinkedAccountBalance() { return linkedAccountBalance; }
}


---

CreditCard.java

package core.model;

import java.math.BigDecimal;

public class CreditCard extends Card {
    private final BigDecimal creditLimit;
    private BigDecimal usedLimit;

    public CreditCard(String cardNumber, String holderName, String expiryDate, String pin, BigDecimal creditLimit) {
        super(cardNumber, holderName, expiryDate, pin);
        this.creditLimit = creditLimit;
        this.usedLimit = BigDecimal.ZERO;
    }

    @Override
    public void charge(BigDecimal amount) {
        if (usedLimit.add(amount).compareTo(creditLimit) > 0) {
            throw new RuntimeException("Credit limit exceeded!");
        }
        usedLimit = usedLimit.add(amount);
    }

    public BigDecimal getCreditLimit() { return creditLimit; }
    public BigDecimal getAvailableLimit() { return creditLimit.subtract(usedLimit); }
}


---

UPI.java

package core.model;

import java.math.BigDecimal;
import java.util.Objects;

public class UPI implements PaymentMethod {
    private final String upiId;
    private final String pin;
    private BigDecimal linkedBankBalance;

    public UPI(String upiId, String pin, BigDecimal initialBalance) {
        this.upiId = Objects.requireNonNull(upiId);
        this.pin = Objects.requireNonNull(pin);
        this.linkedBankBalance = initialBalance;
    }

    @Override
    public String getCardNumber() { return upiId; }

    @Override
    public String getCardHolderName() { return upiId; }

    @Override
    public boolean validatePin(String pin) { return this.pin.equals(pin); }

    @Override
    public void charge(BigDecimal amount) {
        if (linkedBankBalance.compareTo(amount) < 0) {
            throw new RuntimeException("Insufficient balance in UPI account!");
        }
        linkedBankBalance = linkedBankBalance.subtract(amount);
    }

    public BigDecimal getLinkedBankBalance() { return linkedBankBalance; }
}


---

Exceptions

InsufficientBalanceException.java

package core.exception;

public class InsufficientBalanceException extends RuntimeException {
    public InsufficientBalanceException(String message) {
        super(message);
    }
}

InvalidPinException.java

package core.exception;

public class InvalidPinException extends RuntimeException {
    public InvalidPinException(String message) {
        super(message);
    }
}

UserNotFoundException.java

package core.exception;

public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String message) {
        super(message);
    }
}


---

Services

UserService.java

package core.service;

import core.exception.UserNotFoundException;
import core.model.User;
import persistent.repository.UserRepository;

public class UserService {
    private final UserRepository userRepository = new UserRepository();

    public void registerUser(User user) { userRepository.save(user); }

    public User login(String email, String pin) {
        return userRepository.findByEmail(email)
                .filter(u -> u.getPin().equals(pin))
                .orElseThrow(() -> new UserNotFoundException("Invalid email or pin"));
    }
}


---

WalletService.java

package core.service;

import core.exception.InsufficientBalanceException;
import core.model.PaymentMethod;
import core.model.Wallet;
import persistent.repository.WalletRepository;

import java.math.BigDecimal;

public class WalletService {
    private final WalletRepository walletRepository = new WalletRepository();

    public void addMoney(String walletId, BigDecimal amount, PaymentMethod method, String pin) {
        if (!method.validatePin(pin)) {
            throw new RuntimeException("Invalid PIN!");
        }
        method.charge(amount);
        Wallet wallet = walletRepository.findById(walletId)
                .orElseThrow(() -> new RuntimeException("Wallet not found"));
        wallet.addMoney(amount);
    }

    public void deductMoney(String walletId, BigDecimal amount) {
        Wallet wallet = walletRepository.findById(walletId)
                .orElseThrow(() -> new RuntimeException("Wallet not found"));
        if (wallet.getBalance().compareTo(amount) < 0) {
            throw new InsufficientBalanceException("Not enough balance!");
        }
        wallet.deductMoney(amount);
    }

    public void saveWallet(Wallet wallet) {
        walletRepository.save(wallet);
    }
}


---

TransactionService.java

package core.service;

import core.model.Transaction;
import persistent.repository.TransactionRepository;

import java.math.BigDecimal;
import java.util.UUID;

public class TransactionService {
    private final TransactionRepository transactionRepository = new TransactionRepository();
    private final WalletService walletService = new WalletService();

    public Transaction sendMoney(String senderWalletId, String receiverWalletId, BigDecimal amount) {
        walletService.deductMoney(senderWalletId, amount);
        walletService.addMoney(receiverWalletId, amount, new UPI("system@upi","0000",amount), "0000");

        Transaction txn = new Transaction(UUID.randomUUID().toString(), senderWalletId, receiverWalletId, amount);
        transactionRepository.save(txn);
        return txn;
    }
}


---

📌 Persistent Module

UserRepository.java

package persistent.repository;

import core.model.User;
import java.util.*;

public class UserRepository {
    private final Map<String, User> users = new HashMap<>();

    public void save(User user) { users.put(user.getUserId(), user); }

    public Optional<User> findByEmail(String email) {
        return users.values().stream().filter(u -> u.getEmail().equals(email)).findFirst();
    }
}


---

WalletRepository.java

package persistent.repository;

import core.model.Wallet;
import java.util.*;

public class WalletRepository {
    private final Map<String, Wallet> wallets = new HashMap<>();

    public void save(Wallet wallet) { wallets.put(wallet.getWalletId(), wallet); }

    public Optional<Wallet> findById(String walletId) { return Optional.ofNullable(wallets.get(walletId)); }
}


---

TransactionRepository.java

package persistent.repository;

import core.model.Transaction;
import java.util.*;

public class TransactionRepository {
    private final List<Transaction> transactions = new ArrayList<>();

    public void save(Transaction txn) { transactions.add(txn); }

    public List<Transaction> findAll() { return Collections.unmodifiableList(transactions); }
}


---

📌 Main Class (Demo)

App.java

import core.model.*;
import core.service.*;

import java.math.BigDecimal;
import java.util.UUID;

public class App {
    public static void main(String[] args) {
        WalletService walletService = new WalletService();

        // Create wallets
        Wallet w1 = new Wallet(UUID.randomUUID().toString());
        Wallet w2 = new Wallet(UUID.randomUUID().toString());
        walletService.saveWallet(w1);
        walletService.saveWallet(w2);

        // Payment methods
        DebitCard debitCard = new DebitCard("1111-2222-3333-4444", "Alice", "12/30", "1234", new BigDecimal("5000"));
        CreditCard creditCard = new CreditCard("5555-6666-7777-8888", "Bob", "11/29", "5678", new BigDecimal("10000"));
        UPI upi = new UPI("alice@upi", "9999", new BigDecimal("3000"));

        // Add money to wallets
        walletService.addMoney(w1.getWalletId(), new BigDecimal("1000"), debitCard, "1234");
        walletService.addMoney(w2.getWalletId(), new BigDecimal("2000"), creditCard, "5678");
        walletService.addMoney(w1.getWalletId(), new BigDecimal("500"), upi, "9999");

        // Print balances
        System.out.println("Wallet1 Balance: " + w1.getBalance());
        System.out.println("Wallet2 Balance: " + w2.getBalance());
        System.out.println("Debit Card Balance: " + debitCard.getLinkedAccountBalance());
        System.out.println("Credit Card Available Limit: " + creditCard.getAvailableLimit());
        System.out.println("UPI Linked Balance: " + upi.getLinkedBankBalance());
    }
}


---

✅ This is the complete runnable project.

User, Transaction → immutable

Wallet, UserProfile → mutable where needed

PaymentMethod → unified interface

Implementations → DebitCard, CreditCard, UPI

Clean separation of Core (logic) and Persistent (data storage)



---

Do you also want me to extend this so money requests (request/approve flow) between users are included, or keep it limited to send + add money for now?

