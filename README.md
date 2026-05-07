# Online-Exam-System
Online Test System in Java
// OnlineTestSystemGUI.java
// HOW TO RUN:
//   1. Save this file as: OnlineTestSystemGUI.java
//   2. Open terminal in the same folder
//   3. Compile:  javac OnlineTestSystemGUI.java
//   4. Run:      java OnlineTestSystemGUI
//
// Requirements: Java 8 or later (no external libraries needed)

import javax.swing.*;
import javax.swing.border.*;
import javax.swing.table.*;
import java.awt.*;
import java.awt.event.*;
import java.io.*;
import java.util.*;
import java.util.List;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class OnlineTestSystemGUI extends JFrame {

    // ── Data ────────────────────────────────────────────────────────────────
    private static List<User>   users   = new ArrayList<>();
    private static List<Test>   tests   = new ArrayList<>();
    private static List<Result> results = new ArrayList<>();
    private static User currentUser = null;

    private static final String DATA_DIR    = "test_system_data";
    private static final String USERS_FILE  = DATA_DIR + "/users.txt";
    private static final String TESTS_FILE  = DATA_DIR + "/tests.txt";
    private static final String RESULTS_FILE= DATA_DIR + "/results.txt";

    // ── UI ──────────────────────────────────────────────────────────────────
    private CardLayout cardLayout;
    private JPanel     mainPanel;

    // BUG FIX #1 – store dashboard panel references directly instead of
    // looking them up with fragile getComponent(2) / getComponent(3) calls.
    private JPanel studentDashboardPanel;
    private JPanel adminDashboardPanel;

    // ── Constructor ─────────────────────────────────────────────────────────
    public OnlineTestSystemGUI() {
        createDataDirectory();
        loadData();
        initGUI();
        setVisible(true);
    }

    // ════════════════════════════════════════════════════════════════════════
    //  DATA LAYER
    // ════════════════════════════════════════════════════════════════════════

    private void createDataDirectory() {
        File dir = new File(DATA_DIR);
        if (!dir.exists()) dir.mkdir();
    }

    private void loadData() {
        loadUsers();
        loadTests();
        loadResults();
    }

    // ── Users ────────────────────────────────────────────────────────────────
    private void loadUsers() {
        File file = new File(USERS_FILE);
        if (!file.exists()) {
            users.add(new User("admin", "admin@testsystem.com", "admin123", "Administrator", "admin"));
            users.add(new User("john",  "john@example.com",    "pass123",  "John Doe",       "student"));
            users.add(new User("emma",  "emma@example.com",    "pass456",  "Emma Smith",      "student"));
            saveUsers();
            return;
        }
        try (BufferedReader reader = new BufferedReader(new FileReader(USERS_FILE))) {
            String line;
            while ((line = reader.readLine()) != null) {
                String[] parts = line.split("\\|", -1);
                if (parts.length == 5) {
                    users.add(new User(
                        unescape(parts[0]), unescape(parts[1]),
                        unescape(parts[2]), unescape(parts[3]),
                        unescape(parts[4])
                    ));
                }
            }
        } catch (IOException e) { e.printStackTrace(); }
    }

    private void saveUsers() {
        try (BufferedWriter w = new BufferedWriter(new FileWriter(USERS_FILE))) {
            for (User u : users) {
                w.write(escape(u.getUsername()) + "|" +
                        escape(u.getEmail())    + "|" +
                        escape(u.getPassword()) + "|" +
                        escape(u.getFullName()) + "|" +
                        escape(u.getRole()));
                w.newLine();
            }
        } catch (IOException e) { e.printStackTrace(); }
    }

    // ── Tests ────────────────────────────────────────────────────────────────
    private void loadTests() {
        File file = new File(TESTS_FILE);
        if (!file.exists()) { createDefaultTests(); return; }

        try (BufferedReader reader = new BufferedReader(new FileReader(TESTS_FILE))) {
            String line;
            while ((line = reader.readLine()) != null) {
                String[] parts = line.split("\\|", -1);
                if (parts.length >= 3) {
                    Test test = new Test(
                        unescape(parts[0]),
                        unescape(parts[1]),
                        safeParseInt(parts[2], 30)
                    );
                    for (int i = 3; i < parts.length; i++) {
                        // BUG FIX #3 – use a safe separator that won't clash with user text
                        String[] qParts = parts[i].split("~~", -1);
                        if (qParts.length == 6) {
                            List<String> options = new ArrayList<>();
                            // Options separated by "||" (double pipe) to avoid single-pipe clash
                            for (String opt : qParts[2].split("\\|\\|", -1)) {
                                options.add(unescape(opt));
                            }
                            test.addQuestion(new Question(
                                unescape(qParts[0]),
                                unescape(qParts[1]),
                                options,
                                safeParseInt(qParts[3], 1),
                                unescape(qParts[4]),
                                safeParseInt(qParts[5], 10)
                            ));
                        }
                    }
                    tests.add(test);
                }
            }
        } catch (IOException e) { e.printStackTrace(); }
    }

    private void createDefaultTests() {
        Test t1 = new Test("T001", "Java Programming Basics", 30);
        t1.addQuestion(new Question("Q1", "What is the correct main method signature in Java?",
            Arrays.asList("public void main(String[] args)",
                          "public static void main(String[] args)",
                          "public static main(String[] args)",
                          "void main()"),
            2, "The main method must be public, static, void, and accept String[] args", 10));
        t1.addQuestion(new Question("Q2", "Which is NOT a primitive data type in Java?",
            Arrays.asList("int","float","String","boolean"),
            3, "String is a class in Java, not a primitive type", 10));
        t1.addQuestion(new Question("Q3", "What does OOP stand for?",
            Arrays.asList("Object-Oriented Programming","Order of Programming",
                          "Object Organization Protocol","Online Object Protocol"),
            1, "OOP stands for Object-Oriented Programming", 10));
        t1.addQuestion(new Question("Q4", "Which keyword is used to create a class in Java?",
            Arrays.asList("class","interface","package","All of the above"),
            1, "'class' keyword is used to define a class", 10));
        tests.add(t1);

        Test t2 = new Test("T002", "General Knowledge", 15);
        t2.addQuestion(new Question("Q5", "Which is the largest continent?",
            Arrays.asList("Asia","Africa","Europe","Australia"),
            1, "Asia is the largest continent by area", 10));
        t2.addQuestion(new Question("Q6", "Which is the largest ocean?",
            Arrays.asList("Pacific Ocean","Atlantic Ocean","Indian Ocean","Arctic Ocean"),
            1, "Pacific Ocean is the largest ocean on Earth", 10));
        tests.add(t2);

        saveTests();
    }

    private void saveTests() {
        try (BufferedWriter w = new BufferedWriter(new FileWriter(TESTS_FILE))) {
            for (Test t : tests) {
                StringBuilder line = new StringBuilder();
                line.append(escape(t.getId())).append("|")
                    .append(escape(t.getTitle())).append("|")
                    .append(t.getDuration());
                for (Question q : t.getQuestions()) {
                    line.append("|");
                    // Fields separated by ~~ ; options separated by ||
                    line.append(escape(q.getId())).append("~~")
                        .append(escape(q.getText())).append("~~");
                    StringBuilder opts = new StringBuilder();
                    for (int i = 0; i < q.getOptions().size(); i++) {
                        if (i > 0) opts.append("||");
                        opts.append(escape(q.getOptions().get(i)));
                    }
                    line.append(opts)
                        .append("~~").append(q.getCorrectOption())
                        .append("~~").append(escape(q.getExplanation()))
                        .append("~~").append(q.getMarks());
                }
                w.write(line.toString());
                w.newLine();
            }
        } catch (IOException e) { e.printStackTrace(); }
    }

    // ── Results ──────────────────────────────────────────────────────────────
    private void loadResults() {
        File file = new File(RESULTS_FILE);
        if (!file.exists()) return;

        try (BufferedReader reader = new BufferedReader(new FileReader(RESULTS_FILE))) {
            String line;
            while ((line = reader.readLine()) != null) {
                String[] parts = line.split("\\|", -1);
                // BUG FIX: be lenient – accept lines with 5 or 6 parts
                if (parts.length >= 5) {
                    Map<String, Integer> answers = new HashMap<>();
                    String answersPart = parts.length >= 5 ? parts[4] : "";
                    if (!answersPart.isEmpty() && !answersPart.equals("null")) {
                        for (String pair : answersPart.split(",")) {
                            String[] kv = pair.split(":");
                            if (kv.length == 2) {
                                try { answers.put(kv[0], Integer.parseInt(kv[1])); }
                                catch (NumberFormatException ignored) {}
                            }
                        }
                    }
                    String date = parts.length >= 6 ? parts[5] : "";
                    results.add(new Result(parts[0], parts[1], parts[2],
                        safeParseInt(parts[3], 0), answers, date));
                } else {
                    System.err.println("Skipping malformed result line: " + line);
                }
            }
        } catch (IOException e) { e.printStackTrace(); }
    }

    private void saveResults() {
        try (BufferedWriter w = new BufferedWriter(new FileWriter(RESULTS_FILE))) {
            for (Result r : results) {
                StringBuilder ans = new StringBuilder();
                if (r.getAnswers() != null) {
                    boolean first = true;
                    for (Map.Entry<String, Integer> e : r.getAnswers().entrySet()) {
                        if (!first) ans.append(",");
                        ans.append(e.getKey()).append(":").append(e.getValue());
                        first = false;
                    }
                }
                w.write(r.getResultId()    + "|" +
                        r.getStudentId()   + "|" +
                        r.getTestId()      + "|" +
                        r.getScore()       + "|" +
                        ans                + "|" +
                        r.getSubmittedDate());
                w.newLine();
            }
        } catch (IOException e) { e.printStackTrace(); }
    }

    // ── Helpers ──────────────────────────────────────────────────────────────
    // BUG FIX #3 – escape/unescape so pipe and tilde characters in user text
    // don't corrupt the file format.
    private static String escape(String s) {
        if (s == null) return "";
        return s.replace("\\", "\\\\")
                .replace("|",  "\\p")
                .replace("~",  "\\t");
    }
    private static String unescape(String s) {
        if (s == null) return "";
        return s.replace("\\t", "~")
                .replace("\\p", "|")
                .replace("\\\\", "\\");
    }
    private static int safeParseInt(String s, int def) {
        try { return Integer.parseInt(s.trim()); }
        catch (NumberFormatException e) { return def; }
    }

    // ════════════════════════════════════════════════════════════════════════
    //  GUI INIT
    // ════════════════════════════════════════════════════════════════════════

    private void initGUI() {
        setTitle("Online Test System");
        setSize(1000, 700);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setLocationRelativeTo(null);

        cardLayout = new CardLayout();
        mainPanel  = new JPanel(cardLayout);
        mainPanel.setBackground(new Color(240, 242, 245));

        mainPanel.add(createLoginPanel(),    "login");
        mainPanel.add(createRegisterPanel(), "register");

        // BUG FIX #1 – keep explicit references; never use getComponent(n)
        studentDashboardPanel = createStudentDashboard();
        adminDashboardPanel   = createAdminDashboard();
        mainPanel.add(studentDashboardPanel, "studentDashboard");
        mainPanel.add(adminDashboardPanel,   "adminDashboard");

        add(mainPanel);
        cardLayout.show(mainPanel, "login");
    }

    // ════════════════════════════════════════════════════════════════════════
    //  LOGIN PANEL
    // ════════════════════════════════════════════════════════════════════════

    private JPanel createLoginPanel() {
        JPanel panel = new JPanel(new GridBagLayout());
        panel.setBackground(new Color(102, 126, 234));

        JPanel card = new JPanel(new GridBagLayout());
        card.setBackground(Color.WHITE);
        card.setPreferredSize(new Dimension(450, 450));
        card.setBorder(BorderFactory.createCompoundBorder(
            BorderFactory.createLineBorder(new Color(200,200,200)),
            BorderFactory.createEmptyBorder(30,40,30,40)
        ));

        GridBagConstraints gbc = new GridBagConstraints();
        gbc.fill = GridBagConstraints.HORIZONTAL;
        gbc.insets = new Insets(10,0,10,0);

        JLabel title = new JLabel("Online Test System");
        title.setFont(new Font("Segoe UI", Font.BOLD, 24));
        title.setForeground(new Color(102,126,234));
        title.setHorizontalAlignment(SwingConstants.CENTER);
        gbc.gridy = 0; card.add(title, gbc);

        JLabel sub = new JLabel("Login to your account");
        sub.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        sub.setForeground(Color.GRAY);
        sub.setHorizontalAlignment(SwingConstants.CENTER);
        gbc.gridy = 1; card.add(sub, gbc);

        gbc.insets = new Insets(15,0,4,0);
        JLabel ulbl = new JLabel("Username");
        ulbl.setFont(new Font("Segoe UI", Font.BOLD, 13));
        gbc.gridy = 2; card.add(ulbl, gbc);

        JTextField usernameField = styledField();
        gbc.gridy = 3; card.add(usernameField, gbc);

        JLabel plbl = new JLabel("Password");
        plbl.setFont(new Font("Segoe UI", Font.BOLD, 13));
        gbc.gridy = 4; card.add(plbl, gbc);

        JPasswordField passwordField = styledPasswordField();
        gbc.gridy = 5; card.add(passwordRowPanel(passwordField), gbc);

        JButton loginBtn = primaryButton("Login");
        loginBtn.addActionListener(e -> doLogin(
            usernameField.getText(),
            new String(passwordField.getPassword())
        ));
        gbc.gridy = 6; gbc.insets = new Insets(20,0,10,0);
        card.add(loginBtn, gbc);

        JPanel linkRow = new JPanel(new FlowLayout());
        linkRow.setBackground(Color.WHITE);
        linkRow.add(new JLabel("Don't have an account? "));
        JLabel reg = linkLabel("Register");
        reg.addMouseListener(new MouseAdapter() {
            public void mouseClicked(MouseEvent e) { cardLayout.show(mainPanel,"register"); }
        });
        linkRow.add(reg);
        gbc.gridy = 7; card.add(linkRow, gbc);

        panel.add(card);
        return panel;
    }

    private void doLogin(String username, String password) {
        for (User u : users) {
            if (u.getUsername().equals(username) && u.getPassword().equals(password)) {
                currentUser = u;
                JOptionPane.showMessageDialog(this, "Login successful!\nWelcome " + u.getFullName());
                if ("admin".equals(u.getRole())) {
                    refreshAdminDashboard();
                    cardLayout.show(mainPanel, "adminDashboard");
                } else {
                    refreshStudentDashboard();
                    cardLayout.show(mainPanel, "studentDashboard");
                }
                return;
            }
        }
        JOptionPane.showMessageDialog(this,
            "Invalid username or password!", "Error", JOptionPane.ERROR_MESSAGE);
    }

    // ════════════════════════════════════════════════════════════════════════
    //  REGISTER PANEL
    // ════════════════════════════════════════════════════════════════════════

    private JPanel createRegisterPanel() {
        JPanel panel = new JPanel(new GridBagLayout());
        panel.setBackground(new Color(102, 126, 234));

        JPanel card = new JPanel(new GridBagLayout());
        card.setBackground(Color.WHITE);
        card.setPreferredSize(new Dimension(450, 560));
        card.setBorder(BorderFactory.createCompoundBorder(
            BorderFactory.createLineBorder(new Color(200,200,200)),
            BorderFactory.createEmptyBorder(30,40,30,40)
        ));

        GridBagConstraints gbc = new GridBagConstraints();
        gbc.fill = GridBagConstraints.HORIZONTAL;
        gbc.insets = new Insets(6,0,6,0);

        JLabel title = new JLabel("Register");
        title.setFont(new Font("Segoe UI", Font.BOLD, 24));
        title.setForeground(new Color(102,126,234));
        title.setHorizontalAlignment(SwingConstants.CENTER);
        gbc.gridy = 0; card.add(title, gbc);

        gbc.gridy = 1; card.add(boldLabel("Username"), gbc);
        JTextField usernameField = styledField();
        gbc.gridy = 2; card.add(usernameField, gbc);

        gbc.gridy = 3; card.add(boldLabel("Full Name"), gbc);
        JTextField nameField = styledField();
        gbc.gridy = 4; card.add(nameField, gbc);

        gbc.gridy = 5; card.add(boldLabel("Email"), gbc);
        JTextField emailField = styledField();
        gbc.gridy = 6; card.add(emailField, gbc);

        gbc.gridy = 7; card.add(boldLabel("Password"), gbc);
        JPasswordField passwordField = styledPasswordField();
        gbc.gridy = 8; card.add(passwordRowPanel(passwordField), gbc);

        gbc.gridy = 9; card.add(boldLabel("Role"), gbc);
        JComboBox<String> roleCombo = new JComboBox<>(new String[]{"Student","Admin"});
        roleCombo.setPreferredSize(new Dimension(370, 38));
        gbc.gridy = 10; card.add(roleCombo, gbc);

        JButton btn = primaryButton("Register");
        btn.addActionListener(e -> {
            String username = usernameField.getText().trim();
            String fullName = nameField.getText().trim();
            String email    = emailField.getText().trim();
            String password = new String(passwordField.getPassword());
            String role     = roleCombo.getSelectedItem().toString().toLowerCase();

            if (username.isEmpty() || fullName.isEmpty() || email.isEmpty() || password.isEmpty()) {
                JOptionPane.showMessageDialog(this,
                    "Please fill all fields!", "Error", JOptionPane.ERROR_MESSAGE);
                return;
            }
            for (User u : users) {
                if (u.getUsername().equals(username)) {
                    JOptionPane.showMessageDialog(this,
                        "Username already exists!", "Error", JOptionPane.ERROR_MESSAGE);
                    return;
                }
            }
            users.add(new User(username, email, password, fullName, role));
            saveUsers();
            JOptionPane.showMessageDialog(this, "Registration successful!\nPlease login.");
            cardLayout.show(mainPanel, "login");
        });
        gbc.gridy = 11; gbc.insets = new Insets(18,0,10,0);
        card.add(btn, gbc);

        JPanel linkRow = new JPanel(new FlowLayout());
        linkRow.setBackground(Color.WHITE);
        linkRow.add(new JLabel("Already have an account? "));
        JLabel ll = linkLabel("Login");
        ll.addMouseListener(new MouseAdapter() {
            public void mouseClicked(MouseEvent e) { cardLayout.show(mainPanel,"login"); }
        });
        linkRow.add(ll);
        gbc.gridy = 12; card.add(linkRow, gbc);

        panel.add(card);
        return panel;
    }

    // ════════════════════════════════════════════════════════════════════════
    //  STUDENT DASHBOARD
    // ════════════════════════════════════════════════════════════════════════

    private JPanel createStudentDashboard() {
        JPanel panel = new JPanel(new BorderLayout());
        panel.setBackground(new Color(240,242,245));

        // Header
        JPanel header = new JPanel(new BorderLayout());
        header.setBackground(new Color(102,126,234));
        header.setPreferredSize(new Dimension(1000,80));
        header.setBorder(BorderFactory.createEmptyBorder(15,30,15,30));

        JLabel appTitle = new JLabel("Online Test System");
        appTitle.setFont(new Font("Segoe UI", Font.BOLD, 22));
        appTitle.setForeground(Color.WHITE);
        header.add(appTitle, BorderLayout.WEST);

        JPanel userRow = new JPanel(new FlowLayout(FlowLayout.RIGHT));
        userRow.setOpaque(false);
        // BUG FIX #7 – store the label so refreshStudentDashboard can update the name
        JLabel welcomeLbl = new JLabel("Welcome, Student");
        welcomeLbl.setForeground(Color.WHITE);
        welcomeLbl.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        JButton logoutBtn = ghostButton("Logout");
        logoutBtn.addActionListener(e -> { currentUser = null; cardLayout.show(mainPanel,"login"); });
        userRow.add(welcomeLbl);
        userRow.add(logoutBtn);
        header.add(userRow, BorderLayout.EAST);
        panel.add(header, BorderLayout.NORTH);

        // Content
        JPanel content = new JPanel(new BorderLayout());
        content.setBorder(BorderFactory.createEmptyBorder(20,30,20,30));
        content.setBackground(new Color(240,242,245));

        // Stats row – BUG FIX #1: store value labels via putClientProperty
        JPanel statsPanel = new JPanel(new GridLayout(1,3,20,0));
        statsPanel.setOpaque(false);
        statsPanel.add(createStatCard("Total Tests Taken", "0"));
        statsPanel.add(createStatCard("Average Score",     "0"));
        statsPanel.add(createStatCard("Best Score",        "0"));
        content.add(statsPanel, BorderLayout.NORTH);

        // Available tests list
        JPanel testsPanel = new JPanel(new BorderLayout());
        testsPanel.setOpaque(false);
        testsPanel.setBorder(BorderFactory.createEmptyBorder(20,0,0,0));
        JLabel testsTitle = new JLabel("Available Tests");
        testsTitle.setFont(new Font("Segoe UI", Font.BOLD, 18));
        testsPanel.add(testsTitle, BorderLayout.NORTH);

        JPanel testsListPanel = new JPanel();
        testsListPanel.setLayout(new BoxLayout(testsListPanel, BoxLayout.Y_AXIS));
        testsListPanel.setOpaque(false);

        JScrollPane scroll = new JScrollPane(testsListPanel);
        scroll.setBorder(null);
        scroll.getViewport().setBackground(new Color(240,242,245));
        testsPanel.add(scroll, BorderLayout.CENTER);
        content.add(testsPanel, BorderLayout.CENTER);
        panel.add(content, BorderLayout.CENTER);

        // Store references for refresh
        panel.putClientProperty("welcomeLbl",     welcomeLbl);
        panel.putClientProperty("statsPanel",     statsPanel);
        panel.putClientProperty("testsListPanel", testsListPanel);

        return panel;
    }

    private JPanel createStatCard(String title, String value) {
        JPanel card = new JPanel(new BorderLayout());
        card.setBackground(Color.WHITE);
        card.setBorder(BorderFactory.createCompoundBorder(
            BorderFactory.createLineBorder(new Color(220,220,220)),
            BorderFactory.createEmptyBorder(20,20,20,20)
        ));
        JLabel valueLbl = new JLabel(value);
        valueLbl.setFont(new Font("Segoe UI", Font.BOLD, 32));
        valueLbl.setForeground(new Color(102,126,234));
        valueLbl.setHorizontalAlignment(SwingConstants.CENTER);
        card.add(valueLbl, BorderLayout.CENTER);

        JLabel titleLbl = new JLabel(title);
        titleLbl.setFont(new Font("Segoe UI", Font.PLAIN, 13));
        titleLbl.setForeground(Color.GRAY);
        titleLbl.setHorizontalAlignment(SwingConstants.CENTER);
        card.add(titleLbl, BorderLayout.SOUTH);

        // BUG FIX #1 – store the value label for direct access later
        card.putClientProperty("valueLabel", valueLbl);
        return card;
    }

    private void refreshStudentDashboard() {
        JPanel panel       = studentDashboardPanel;
        JLabel welcomeLbl  = (JLabel) panel.getClientProperty("welcomeLbl");
        JPanel statsPanel  = (JPanel) panel.getClientProperty("statsPanel");
        JPanel testsListPanel = (JPanel) panel.getClientProperty("testsListPanel");

        // BUG FIX #7 – update welcome label with real user name
        if (welcomeLbl != null && currentUser != null)
            welcomeLbl.setText("Welcome, " + currentUser.getFullName());

        // Compute stats for this student
        List<Result> mine = new ArrayList<>();
        for (Result r : results)
            if (r.getStudentId().equals(currentUser.getUsername())) mine.add(r);

        int totalTests = mine.size(), totalScore = 0, best = 0;
        for (Result r : mine) {
            totalScore += r.getScore();
            if (r.getScore() > best) best = r.getScore();
        }
        int avg = totalTests > 0 ? totalScore / totalTests : 0;

        // BUG FIX #1 – use putClientProperty value labels directly, no fragile casting
        Component[] statCards = statsPanel.getComponents();
        setStatValue((JPanel) statCards[0], String.valueOf(totalTests));
        setStatValue((JPanel) statCards[1], String.valueOf(avg));
        setStatValue((JPanel) statCards[2], String.valueOf(best));

        // Rebuild test cards
        testsListPanel.removeAll();
        Set<String> done = new HashSet<>();
        for (Result r : results)
            if (r.getStudentId().equals(currentUser.getUsername())) done.add(r.getTestId());

        for (Test t : tests) {
            testsListPanel.add(createTestCard(t, !done.contains(t.getId())));
            testsListPanel.add(Box.createRigidArea(new Dimension(0, 15)));
        }
        testsListPanel.revalidate();
        testsListPanel.repaint();
    }

    private void setStatValue(JPanel card, String value) {
        JLabel lbl = (JLabel) card.getClientProperty("valueLabel");
        if (lbl != null) lbl.setText(value);
    }

    private JPanel createTestCard(Test test, boolean available) {
        JPanel card = new JPanel(new BorderLayout());
        card.setBackground(Color.WHITE);
        card.setBorder(BorderFactory.createCompoundBorder(
            BorderFactory.createLineBorder(new Color(220,220,220)),
            BorderFactory.createEmptyBorder(15,15,15,15)
        ));
        card.setMaximumSize(new Dimension(800,120));
        card.setPreferredSize(new Dimension(800,110));

        JPanel info = new JPanel(new BorderLayout());
        info.setOpaque(false);
        JLabel ttl = new JLabel(test.getTitle());
        ttl.setFont(new Font("Segoe UI", Font.BOLD, 16));
        info.add(ttl, BorderLayout.NORTH);

        JPanel details = new JPanel(new FlowLayout(FlowLayout.LEFT, 20, 5));
        details.setOpaque(false);
        details.add(new JLabel("Questions: " + test.getQuestions().size()));
        details.add(new JLabel("Duration: " + test.getDuration() + " min"));
        details.add(new JLabel("Total Marks: " + getTotalMarks(test)));
        info.add(details, BorderLayout.CENTER);
        card.add(info, BorderLayout.CENTER);

        if (available) {
            JButton btn = new JButton("Take Test");
            btn.setBackground(new Color(76,175,80));
            btn.setForeground(Color.WHITE);
            btn.setBorderPainted(false);
            btn.setFocusPainted(false);
            btn.setPreferredSize(new Dimension(120, 35));
            btn.addActionListener(e -> startTest(test));
            card.add(btn, BorderLayout.EAST);
        } else {
            JLabel done = new JLabel("Completed");
            done.setForeground(new Color(76,175,80));
            done.setFont(new Font("Segoe UI", Font.BOLD, 14));
            JPanel p = new JPanel();
            p.setOpaque(false);
            p.add(done);
            card.add(p, BorderLayout.EAST);
        }
        return card;
    }

    // ════════════════════════════════════════════════════════════════════════
    //  TEST TAKING
    // ════════════════════════════════════════════════════════════════════════

    private void startTest(Test test) {
        JFrame tf = new JFrame("Taking Test: " + test.getTitle());
        tf.setSize(800, 600);
        tf.setLocationRelativeTo(this);
        // BUG FIX #4 – prevent OS close button from silently leaving timer running
        tf.setDefaultCloseOperation(JFrame.DO_NOTHING_ON_CLOSE);

        JPanel main = new JPanel(new BorderLayout());

        JPanel timerBar = new JPanel();
        timerBar.setBackground(new Color(102,126,234));
        JLabel timerLbl = new JLabel(String.format("Time Remaining: %02d:00", test.getDuration()));
        timerLbl.setFont(new Font("Segoe UI", Font.BOLD, 18));
        timerLbl.setForeground(Color.WHITE);
        timerBar.add(timerLbl);
        main.add(timerBar, BorderLayout.NORTH);

        JPanel questionsPanel = new JPanel();
        questionsPanel.setLayout(new BoxLayout(questionsPanel, BoxLayout.Y_AXIS));
        questionsPanel.setBorder(BorderFactory.createEmptyBorder(20,20,20,20));

        List<Integer> answers = new ArrayList<>(Collections.nCopies(test.getQuestions().size(), 0));

        for (int i = 0; i < test.getQuestions().size(); i++) {
            Question q = test.getQuestions().get(i);
            JPanel qp = new JPanel(new BorderLayout());
            qp.setBorder(BorderFactory.createCompoundBorder(
                BorderFactory.createLineBorder(new Color(220,220,220)),
                BorderFactory.createEmptyBorder(10,10,10,10)
            ));
            qp.setBackground(Color.WHITE);

            JLabel ql = new JLabel("<html><b>Q" + (i+1) + ".</b> " + q.getText() +
                " <span style='color:#667eea;'>[" + q.getMarks() + " marks]</span></html>");
            qp.add(ql, BorderLayout.NORTH);

            JPanel opts = new JPanel(new GridLayout(4,1,5,5));
            opts.setBorder(BorderFactory.createEmptyBorder(10,20,0,0));
            opts.setOpaque(false);
            ButtonGroup grp = new ButtonGroup();
            final int qi = i;
            for (int j = 0; j < q.getOptions().size(); j++) {
                JRadioButton rb = new JRadioButton(q.getOptions().get(j));
                final int oi = j + 1;
                rb.addActionListener(e -> answers.set(qi, oi));
                grp.add(rb);
                opts.add(rb);
            }
            qp.add(opts, BorderLayout.CENTER);
            questionsPanel.add(qp);
            questionsPanel.add(Box.createRigidArea(new Dimension(0,15)));
        }

        main.add(new JScrollPane(questionsPanel), BorderLayout.CENTER);

        JButton submitBtn = new JButton("Submit Test");
        submitBtn.setBackground(new Color(76,175,80));
        submitBtn.setForeground(Color.WHITE);
        submitBtn.setFont(new Font("Segoe UI", Font.BOLD, 14));
        submitBtn.setFocusPainted(false);
        submitBtn.setBorderPainted(false);
        JPanel sp = new JPanel();
        sp.add(submitBtn);
        main.add(sp, BorderLayout.SOUTH);

        tf.add(main);
        tf.setVisible(true);

        // Timer
        javax.swing.Timer[] timerRef = new javax.swing.Timer[1];
        timerRef[0] = new javax.swing.Timer(1000, new ActionListener() {
            int left = test.getDuration() * 60;
            public void actionPerformed(ActionEvent e) {
                left--;
                timerLbl.setText(String.format("Time Remaining: %02d:%02d", left/60, left%60));
                if (left <= 0) {
                    timerRef[0].stop();
                    calculateAndShowResult(test, answers, tf);
                }
            }
        });
        timerRef[0].start();

        submitBtn.addActionListener(e -> {
            timerRef[0].stop();
            calculateAndShowResult(test, answers, tf);
        });

        // BUG FIX #4 – OS close button stops the timer cleanly
        tf.addWindowListener(new WindowAdapter() {
            public void windowClosing(WindowEvent e) {
                timerRef[0].stop();
                tf.dispose();
            }
        });
    }

    private void calculateAndShowResult(Test test, List<Integer> answers, JFrame tf) {
        int totalScore = 0, correct = 0;
        Map<String, Integer> answerMap = new HashMap<>();

        for (int i = 0; i < test.getQuestions().size(); i++) {
            Question q = test.getQuestions().get(i);
            int ua = answers.get(i);
            if (ua > 0) answerMap.put(q.getId(), ua);
            if (ua == q.getCorrectOption()) { totalScore += q.getMarks(); correct++; }
        }

        String resultId  = "RES_" + System.currentTimeMillis();
        String submitted = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        results.add(new Result(resultId, currentUser.getUsername(), test.getId(),
            totalScore, answerMap, submitted));
        saveResults();
        tf.dispose();

        // Build result message
        int total = getTotalMarks(test);
        StringBuilder msg = new StringBuilder();
        msg.append("═══════════════════════════════════\n");
        msg.append("          TEST COMPLETED!\n");
        msg.append("═══════════════════════════════════\n\n");
        msg.append("Your Score : ").append(totalScore).append(" / ").append(total).append("\n");
        msg.append("Correct    : ").append(correct).append(" / ").append(test.getQuestions().size()).append("\n");
        msg.append(String.format("Percentage : %.2f%%\n\n", total > 0 ? (double)totalScore/total*100 : 0));
        msg.append("Detailed Analysis:\n──────────────────\n");
        for (int i = 0; i < test.getQuestions().size(); i++) {
            Question q  = test.getQuestions().get(i);
            int ua      = answers.get(i);
            msg.append("\nQ").append(i+1).append(". ").append(q.getText()).append("\n");
            if (ua == 0)                       msg.append("   Not attempted\n");
            else if (ua == q.getCorrectOption()) msg.append("   Correct!\n");
            else {
                msg.append("   Incorrect – correct answer: ")
                   .append(q.getOptions().get(q.getCorrectOption()-1)).append("\n");
            }
            msg.append("   Explanation: ").append(q.getExplanation()).append("\n");
        }

        JTextArea ta = new JTextArea(msg.toString());
        ta.setEditable(false);
        ta.setFont(new Font("Monospaced", Font.PLAIN, 12));
        JScrollPane sp = new JScrollPane(ta);
        sp.setPreferredSize(new Dimension(600, 500));
        JOptionPane.showMessageDialog(this, sp, "Test Results", JOptionPane.INFORMATION_MESSAGE);

        refreshStudentDashboard();
    }

    private int getTotalMarks(Test test) {
        int t = 0;
        for (Question q : test.getQuestions()) t += q.getMarks();
        return t;
    }

    // ════════════════════════════════════════════════════════════════════════
    //  ADMIN DASHBOARD
    // ════════════════════════════════════════════════════════════════════════

    private JPanel createAdminDashboard() {
        JPanel panel = new JPanel(new BorderLayout());
        panel.setBackground(new Color(240,242,245));

        JPanel header = new JPanel(new BorderLayout());
        header.setBackground(new Color(102,126,234));
        header.setPreferredSize(new Dimension(1000,80));
        header.setBorder(BorderFactory.createEmptyBorder(15,30,15,30));
        JLabel ttl = new JLabel("Online Test System – Admin Dashboard");
        ttl.setFont(new Font("Segoe UI", Font.BOLD, 22));
        ttl.setForeground(Color.WHITE);
        header.add(ttl, BorderLayout.WEST);

        JPanel urp = new JPanel(new FlowLayout(FlowLayout.RIGHT));
        urp.setOpaque(false);
        JButton lb = ghostButton("Logout");
        lb.addActionListener(e -> { currentUser = null; cardLayout.show(mainPanel,"login"); });
        urp.add(lb);
        header.add(urp, BorderLayout.EAST);
        panel.add(header, BorderLayout.NORTH);

        JPanel mc = new JPanel(new BorderLayout());
        mc.setBorder(BorderFactory.createEmptyBorder(20,20,20,20));
        mc.setBackground(new Color(240,242,245));

        // Sidebar
        JPanel sidebar = new JPanel();
        sidebar.setLayout(new BoxLayout(sidebar, BoxLayout.Y_AXIS));
        sidebar.setBackground(Color.WHITE);
        sidebar.setPreferredSize(new Dimension(220,500));
        sidebar.setBorder(BorderFactory.createLineBorder(new Color(220,220,220)));

        for (String item : new String[]{"Create New Test","View All Tests","View All Users","View All Results"}) {
            JButton btn = new JButton(item);
            btn.setAlignmentX(Component.CENTER_ALIGNMENT);
            btn.setMaximumSize(new Dimension(190,45));
            btn.setBackground(Color.WHITE);
            btn.setForeground(new Color(80,80,80));
            btn.setFont(new Font("Segoe UI", Font.PLAIN, 14));
            btn.setBorder(BorderFactory.createCompoundBorder(
                BorderFactory.createLineBorder(new Color(220,220,220)),
                BorderFactory.createEmptyBorder(10,10,10,10)
            ));
            btn.setFocusPainted(false);
            btn.addActionListener(e -> {
                switch (item) {
                    case "Create New Test":  showCreateTestDialog(); break;
                    case "View All Tests":   showAllTestsDialog();   break;
                    case "View All Users":   showAllUsersDialog();   break;
                    case "View All Results": showAllResultsDialog(); break;
                }
            });
            sidebar.add(Box.createRigidArea(new Dimension(0,10)));
            sidebar.add(btn);
        }
        mc.add(sidebar, BorderLayout.WEST);

        // Content area
        JPanel contentArea = new JPanel(new BorderLayout());
        contentArea.setBackground(Color.WHITE);
        contentArea.setBorder(BorderFactory.createEmptyBorder(20,20,20,20));
        panel.putClientProperty("contentArea", contentArea);

        JLabel ct = new JLabel("All Tests");
        ct.setFont(new Font("Segoe UI", Font.BOLD, 20));
        contentArea.add(ct, BorderLayout.NORTH);

        JPanel tl = buildAdminTestListPanel();
        contentArea.add(new JScrollPane(tl), BorderLayout.CENTER);

        mc.add(contentArea, BorderLayout.CENTER);
        panel.add(mc, BorderLayout.CENTER);

        return panel;
    }

    private JPanel buildAdminTestListPanel() {
        JPanel list = new JPanel();
        list.setLayout(new BoxLayout(list, BoxLayout.Y_AXIS));
        for (Test t : tests) {
            list.add(createAdminTestCard(t));
            list.add(Box.createRigidArea(new Dimension(0,15)));
        }
        return list;
    }

    private void refreshAdminDashboard() {
        JPanel contentArea = (JPanel) adminDashboardPanel.getClientProperty("contentArea");
        contentArea.removeAll();

        JLabel ct = new JLabel("All Tests");
        ct.setFont(new Font("Segoe UI", Font.BOLD, 20));
        contentArea.add(ct, BorderLayout.NORTH);
        contentArea.add(new JScrollPane(buildAdminTestListPanel()), BorderLayout.CENTER);

        contentArea.revalidate();
        contentArea.repaint();
    }

    private JPanel createAdminTestCard(Test test) {
        JPanel card = new JPanel(new BorderLayout());
        card.setBackground(Color.WHITE);
        card.setBorder(BorderFactory.createCompoundBorder(
            BorderFactory.createLineBorder(new Color(220,220,220)),
            BorderFactory.createEmptyBorder(15,15,15,15)
        ));

        JPanel info = new JPanel(new BorderLayout());
        info.setOpaque(false);
        JLabel ttl = new JLabel(test.getTitle());
        ttl.setFont(new Font("Segoe UI", Font.BOLD, 16));
        info.add(ttl, BorderLayout.NORTH);
        JLabel idLbl = new JLabel("ID: " + test.getId());
        idLbl.setForeground(Color.GRAY);
        info.add(idLbl, BorderLayout.CENTER);
        JPanel det = new JPanel(new FlowLayout(FlowLayout.LEFT,20,5));
        det.setOpaque(false);
        det.add(new JLabel("Questions: " + test.getQuestions().size()));
        det.add(new JLabel("Duration: " + test.getDuration() + " min"));
        info.add(det, BorderLayout.SOUTH);
        card.add(info, BorderLayout.CENTER);

        JPanel btns = new JPanel(new FlowLayout());
        btns.setOpaque(false);

        JButton edit = new JButton("Edit");
        edit.setBackground(new Color(255,193,7));
        edit.setForeground(Color.BLACK);
        edit.setBorderPainted(false);
        edit.setFocusPainted(false);
        edit.addActionListener(e -> editTest(test));

        JButton del = new JButton("Delete");
        del.setBackground(new Color(220,53,69));
        del.setForeground(Color.WHITE);
        del.setBorderPainted(false);
        del.setFocusPainted(false);
        del.addActionListener(e -> deleteTest(test));

        btns.add(edit);
        btns.add(del);
        card.add(btns, BorderLayout.EAST);
        return card;
    }

    // ════════════════════════════════════════════════════════════════════════
    //  ADMIN DIALOGS
    // ════════════════════════════════════════════════════════════════════════

    private void showCreateTestDialog() {
        JDialog dlg = new JDialog(this, "Create New Test", true);
        dlg.setSize(500, 380);
        dlg.setLocationRelativeTo(this);

        JPanel panel = new JPanel(new GridBagLayout());
        panel.setBorder(BorderFactory.createEmptyBorder(20,20,20,20));
        GridBagConstraints gbc = new GridBagConstraints();
        gbc.fill = GridBagConstraints.HORIZONTAL;
        gbc.insets = new Insets(8,8,8,8);

        gbc.gridy = 0; panel.add(new JLabel("Test Title:"), gbc);
        JTextField titleField = new JTextField(); gbc.gridy = 1; panel.add(titleField, gbc);

        gbc.gridy = 2; panel.add(new JLabel("Duration (minutes):"), gbc);
        JTextField durField = new JTextField(); gbc.gridy = 3; panel.add(durField, gbc);

        gbc.gridy = 4; panel.add(new JLabel("Number of Questions:"), gbc);
        JSpinner spinner = new JSpinner(new SpinnerNumberModel(1,1,20,1));
        gbc.gridy = 5; panel.add(spinner, gbc);

        JButton create = new JButton("Create Test");
        create.setBackground(new Color(76,175,80));
        create.setForeground(Color.WHITE);
        create.setFocusPainted(false);
        create.setBorderPainted(false);
        gbc.gridy = 6; gbc.insets = new Insets(20,8,8,8);
        panel.add(create, gbc);

        create.addActionListener(e -> {
            String title = titleField.getText().trim();
            // BUG FIX #5 – validate input before parseInt
            if (title.isEmpty() || durField.getText().trim().isEmpty()) {
                JOptionPane.showMessageDialog(dlg, "Please fill all fields!", "Error", JOptionPane.ERROR_MESSAGE);
                return;
            }
            int dur;
            try { dur = Integer.parseInt(durField.getText().trim()); }
            catch (NumberFormatException ex) {
                JOptionPane.showMessageDialog(dlg, "Duration must be a whole number.", "Error", JOptionPane.ERROR_MESSAGE);
                return;
            }
            if (dur <= 0) {
                JOptionPane.showMessageDialog(dlg, "Duration must be greater than 0.", "Error", JOptionPane.ERROR_MESSAGE);
                return;
            }
            int qc = (Integer) spinner.getValue();
            dlg.dispose();
            createTestWithQuestions(title, dur, qc);
        });

        dlg.add(panel);
        dlg.setVisible(true);
    }

    private void createTestWithQuestions(String title, int duration, int qCount) {
        JDialog dlg = new JDialog(this, "Add Questions", true);
        dlg.setSize(620, 520);
        dlg.setLocationRelativeTo(this);

        JPanel root = new JPanel(new BorderLayout());

        JPanel qPanel = new JPanel();
        qPanel.setLayout(new BoxLayout(qPanel, BoxLayout.Y_AXIS));
        qPanel.setBorder(BorderFactory.createEmptyBorder(10,10,10,10));

        List<JTextField>       qFields    = new ArrayList<>();
        List<List<JTextField>> optFields  = new ArrayList<>();
        List<JComboBox<Integer>> corrFields = new ArrayList<>();
        List<JTextField>       markFields = new ArrayList<>();

        for (int i = 0; i < qCount; i++) {
            JPanel qp = new JPanel(new GridBagLayout());
            qp.setBorder(BorderFactory.createTitledBorder("Question " + (i+1)));
            GridBagConstraints g = new GridBagConstraints();
            g.fill = GridBagConstraints.HORIZONTAL; g.insets = new Insets(4,5,4,5);

            g.gridy = 0; qp.add(new JLabel("Question:"), g);
            JTextField qf = new JTextField(40); g.gridy = 1; qp.add(qf, g);
            qFields.add(qf);

            List<JTextField> opts = new ArrayList<>();
            for (int j = 0; j < 4; j++) {
                g.gridy = 2 + j*2; qp.add(new JLabel("Option " + (j+1) + ":"), g);
                JTextField of = new JTextField(30);
                g.gridy = 3 + j*2; qp.add(of, g);
                opts.add(of);
            }
            optFields.add(opts);

            g.gridy = 10; qp.add(new JLabel("Correct Option (1-4):"), g);
            JComboBox<Integer> cc = new JComboBox<>(new Integer[]{1,2,3,4});
            g.gridy = 11; qp.add(cc, g);
            corrFields.add(cc);

            g.gridy = 12; qp.add(new JLabel("Marks:"), g);
            JTextField mf = new JTextField("10", 5);
            g.gridy = 13; qp.add(mf, g);
            markFields.add(mf);

            qPanel.add(qp);
        }

        root.add(new JScrollPane(qPanel), BorderLayout.CENTER);

        JButton save = new JButton("Save Test");
        save.setBackground(new Color(76,175,80));
        save.setForeground(Color.WHITE);
        save.setFocusPainted(false);
        save.setBorderPainted(false);
        JPanel bp = new JPanel(); bp.add(save);
        root.add(bp, BorderLayout.SOUTH);

        String testId   = "T" + System.currentTimeMillis();
        Test   newTest  = new Test(testId, title, duration);

        save.addActionListener(e -> {
            // BUG FIX #5 – validate marks before parseInt
            // BUG FIX #8 – capture timestamp ONCE before the loop
            long ts = System.currentTimeMillis();
            for (int i = 0; i < qCount; i++) {
                String qt = qFields.get(i).getText().trim();
                if (qt.isEmpty()) continue;

                List<String> options = new ArrayList<>();
                for (JTextField of : optFields.get(i)) options.add(of.getText().trim());

                int marks;
                try { marks = Integer.parseInt(markFields.get(i).getText().trim()); }
                catch (NumberFormatException ex) { marks = 10; }

                int correct = (Integer) corrFields.get(i).getSelectedItem();
                String qId  = "Q" + ts + "_" + i;   // unique because ts is fixed & i changes
                newTest.addQuestion(new Question(qId, qt, options, correct,
                    "Correct answer is option " + correct, marks));
            }
            if (newTest.getQuestions().isEmpty()) {
                JOptionPane.showMessageDialog(dlg, "Please add at least one question.");
                return;
            }
            tests.add(newTest);
            saveTests();
            JOptionPane.showMessageDialog(dlg, "Test created successfully!");
            dlg.dispose();
            refreshAdminDashboard();
        });

        dlg.add(root);
        dlg.setVisible(true);
    }

    private void editTest(Test test) {
        JDialog dlg = new JDialog(this, "Edit Test: " + test.getTitle(), true);
        dlg.setSize(400, 280);
        dlg.setLocationRelativeTo(this);

        JPanel panel = new JPanel(new GridBagLayout());
        panel.setBorder(BorderFactory.createEmptyBorder(20,20,20,20));
        GridBagConstraints gbc = new GridBagConstraints();
        gbc.fill = GridBagConstraints.HORIZONTAL;
        gbc.insets = new Insets(8,8,8,8);

        gbc.gridy = 0; panel.add(new JLabel("Test Title:"), gbc);
        JTextField tf = new JTextField(test.getTitle(), 20); gbc.gridy = 1; panel.add(tf, gbc);

        gbc.gridy = 2; panel.add(new JLabel("Duration (minutes):"), gbc);
        JTextField df = new JTextField(String.valueOf(test.getDuration()), 20);
        gbc.gridy = 3; panel.add(df, gbc);

        JButton save = new JButton("Save Changes");
        save.setBackground(new Color(76,175,80));
        save.setForeground(Color.WHITE);
        save.setFocusPainted(false);
        save.setBorderPainted(false);
        gbc.gridy = 4; gbc.insets = new Insets(20,8,8,8);
        panel.add(save, gbc);

        save.addActionListener(e -> {
            String title = tf.getText().trim();
            if (title.isEmpty()) {
                JOptionPane.showMessageDialog(dlg, "Title cannot be empty.");
                return;
            }
            int dur;
            try { dur = Integer.parseInt(df.getText().trim()); }
            catch (NumberFormatException ex) {
                JOptionPane.showMessageDialog(dlg, "Duration must be a whole number.", "Error", JOptionPane.ERROR_MESSAGE);
                return;
            }
            test.setTitle(title);
            test.setDuration(dur);
            saveTests();
            JOptionPane.showMessageDialog(dlg, "Test updated successfully!");
            dlg.dispose();
            refreshAdminDashboard();
        });

        dlg.add(panel);
        dlg.setVisible(true);
    }

    private void deleteTest(Test test) {
        int ok = JOptionPane.showConfirmDialog(this,
            "Delete test '" + test.getTitle() + "'?",
            "Confirm Delete", JOptionPane.YES_NO_OPTION);
        if (ok == JOptionPane.YES_OPTION) {
            tests.remove(test);
            results.removeIf(r -> r.getTestId().equals(test.getId()));
            saveTests();
            saveResults();
            JOptionPane.showMessageDialog(this, "Test deleted.");
            refreshAdminDashboard();
        }
    }

    private void showAllTestsDialog() {
        JDialog dlg = new JDialog(this, "All Tests", true);
        dlg.setSize(800, 500);
        dlg.setLocationRelativeTo(this);

        JPanel p = new JPanel(new BorderLayout());
        p.setBorder(BorderFactory.createEmptyBorder(10,10,10,10));
        JPanel list = new JPanel();
        list.setLayout(new BoxLayout(list, BoxLayout.Y_AXIS));
        for (Test t : tests) {
            list.add(createAdminTestCard(t));
            list.add(Box.createRigidArea(new Dimension(0,10)));
        }
        p.add(new JScrollPane(list), BorderLayout.CENTER);
        dlg.add(p);
        dlg.setVisible(true);
    }

    private void showAllUsersDialog() {
        JDialog dlg = new JDialog(this, "All Users", true);
        dlg.setSize(800, 450);
        dlg.setLocationRelativeTo(this);

        String[] cols = {"Username","Full Name","Email","Role","Tests Taken","Avg Score"};
        Object[][] data = new Object[users.size()][6];
        for (int i = 0; i < users.size(); i++) {
            User u = users.get(i);
            data[i][0] = u.getUsername();
            data[i][1] = u.getFullName();
            data[i][2] = u.getEmail();
            data[i][3] = u.getRole();
            int cnt = 0, tot = 0;
            for (Result r : results)
                if (r.getStudentId().equals(u.getUsername())) { cnt++; tot += r.getScore(); }
            data[i][4] = cnt;
            data[i][5] = cnt > 0 ? tot/cnt : 0;
        }
        JTable tbl = new JTable(data, cols);
        tbl.setRowHeight(30);
        tbl.setFont(new Font("Segoe UI", Font.PLAIN, 13));
        tbl.getTableHeader().setFont(new Font("Segoe UI", Font.BOLD, 13));
        dlg.add(new JScrollPane(tbl));
        dlg.setVisible(true);
    }

    private void showAllResultsDialog() {
        JDialog dlg = new JDialog(this, "All Results", true);
        dlg.setSize(900, 450);
        dlg.setLocationRelativeTo(this);

        String[] cols = {"Result ID","Student","Test ID","Score","Date"};
        Object[][] data = new Object[results.size()][5];
        for (int i = 0; i < results.size(); i++) {
            Result r = results.get(i);
            data[i][0] = r.getResultId();
            data[i][1] = r.getStudentId();
            data[i][2] = r.getTestId();
            data[i][3] = r.getScore();
            data[i][4] = r.getSubmittedDate();
        }
        JTable tbl = new JTable(data, cols);
        tbl.setRowHeight(30);
        tbl.setFont(new Font("Segoe UI", Font.PLAIN, 13));
        tbl.getTableHeader().setFont(new Font("Segoe UI", Font.BOLD, 13));
        dlg.add(new JScrollPane(tbl));
        dlg.setVisible(true);
    }

    // ════════════════════════════════════════════════════════════════════════
    //  UI HELPERS
    // ════════════════════════════════════════════════════════════════════════

    private JTextField styledField() {
        JTextField f = new JTextField();
        f.setPreferredSize(new Dimension(370, 40));
        f.setBorder(BorderFactory.createCompoundBorder(
            BorderFactory.createLineBorder(new Color(220,220,220)),
            BorderFactory.createEmptyBorder(5,10,5,10)
        ));
        return f;
    }

    private JPasswordField styledPasswordField() {
        JPasswordField f = new JPasswordField();
        f.setPreferredSize(new Dimension(370, 40));
        f.setBorder(BorderFactory.createCompoundBorder(
            BorderFactory.createLineBorder(new Color(220,220,220)),
            BorderFactory.createEmptyBorder(5,10,5,10)
        ));
        return f;
    }

    /** Wraps a JPasswordField with a show/hide toggle button on the right. */
    private JPanel passwordRowPanel(JPasswordField passwordField) {
        JPanel wrapper = new JPanel(new BorderLayout(0, 0));
        wrapper.setOpaque(false);
        wrapper.setPreferredSize(new Dimension(370, 40));

        // Make the field fill the panel (remove its own preferred width)
        passwordField.setPreferredSize(null);
        wrapper.add(passwordField, BorderLayout.CENTER);

        JButton toggleBtn = new JButton("\uD83D\uDC41"); // 👁 eye emoji
        toggleBtn.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        toggleBtn.setFocusPainted(false);
        toggleBtn.setBorderPainted(false);
        toggleBtn.setContentAreaFilled(false);
        toggleBtn.setCursor(new Cursor(Cursor.HAND_CURSOR));
        toggleBtn.setToolTipText("Show / Hide password");
        toggleBtn.setPreferredSize(new Dimension(42, 40));
        toggleBtn.setBackground(Color.WHITE);

        // Track visibility state
        final boolean[] visible = {false};
        toggleBtn.addActionListener(e -> {
            visible[0] = !visible[0];
            if (visible[0]) {
                passwordField.setEchoChar((char) 0);   // show plain text
                toggleBtn.setText("\uD83D\uDEC9");     // 🛩 → use a closed-eye substitute
                toggleBtn.setToolTipText("Hide password");
            } else {
                passwordField.setEchoChar('\u2022');   // restore bullet
                toggleBtn.setText("\uD83D\uDC41");     // 👁
                toggleBtn.setToolTipText("Show password");
            }
        });

        wrapper.add(toggleBtn, BorderLayout.EAST);

        // Give wrapper same border style as styledField/styledPasswordField
        wrapper.setBorder(BorderFactory.createCompoundBorder(
            BorderFactory.createLineBorder(new Color(220, 220, 220)),
            BorderFactory.createEmptyBorder(0, 0, 0, 0)
        ));
        // Remove the inner border from the password field itself so borders don't double-up
        passwordField.setBorder(BorderFactory.createEmptyBorder(5, 10, 5, 4));

        return wrapper;
    }

    private JButton primaryButton(String text) {
        JButton btn = new JButton(text);
        btn.setBackground(new Color(102,126,234));
        btn.setForeground(Color.WHITE);
        btn.setFont(new Font("Segoe UI", Font.BOLD, 14));
        btn.setPreferredSize(new Dimension(370, 45));
        btn.setBorderPainted(false);
        btn.setFocusPainted(false);
        return btn;
    }

    private JButton ghostButton(String text) {
        JButton btn = new JButton(text);
        btn.setBackground(new Color(255,255,255,50));
        btn.setForeground(Color.WHITE);
        btn.setBorderPainted(false);
        btn.setFocusPainted(false);
        return btn;
    }

    private JLabel boldLabel(String text) {
        JLabel l = new JLabel(text);
        l.setFont(new Font("Segoe UI", Font.BOLD, 13));
        return l;
    }

    private JLabel linkLabel(String text) {
        JLabel l = new JLabel(text);
        l.setForeground(new Color(102,126,234));
        l.setCursor(new Cursor(Cursor.HAND_CURSOR));
        return l;
    }

    // ════════════════════════════════════════════════════════════════════════
    //  MAIN
    // ════════════════════════════════════════════════════════════════════════

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            try { UIManager.setLookAndFeel(UIManager.getSystemLookAndFeelClassName()); }
            catch (Exception ignored) {}
            new OnlineTestSystemGUI();
        });
    }
}

// ════════════════════════════════════════════════════════════════════════════
//  MODEL CLASSES
// ════════════════════════════════════════════════════════════════════════════

class User {
    private final String username, email, password, fullName, role;
    public User(String username, String email, String password, String fullName, String role) {
        this.username = username; this.email = email; this.password = password;
        this.fullName = fullName; this.role  = role;
    }
    public String getUsername() { return username; }
    public String getEmail()    { return email;    }
    public String getPassword() { return password; }
    public String getFullName() { return fullName; }
    public String getRole()     { return role;     }
}

class Question {
    private final String id, text, explanation;
    private final List<String> options;
    private final int correctOption, marks;
    public Question(String id, String text, List<String> options,
                    int correctOption, String explanation, int marks) {
        this.id = id; this.text = text; this.options = options;
        this.correctOption = correctOption; this.explanation = explanation; this.marks = marks;
    }
    public String       getId()            { return id;            }
    public String       getText()          { return text;          }
    public List<String> getOptions()       { return options;       }
    public int          getCorrectOption() { return correctOption; }
    public String       getExplanation()   { return explanation;   }
    public int          getMarks()         { return marks;         }
}

class Test {
    private String id, title;
    private int duration;
    private final List<Question> questions = new ArrayList<>();
    public Test(String id, String title, int duration) {
        this.id = id; this.title = title; this.duration = duration;
    }
    public String         getId()        { return id;        }
    public String         getTitle()     { return title;     }
    public int            getDuration()  { return duration;  }
    public List<Question> getQuestions() { return questions; }
    public void setTitle(String t)       { this.title = t;   }
    public void setDuration(int d)       { this.duration = d;}
    public void addQuestion(Question q)  { questions.add(q); }
}

class Result {
    private final String resultId, studentId, testId, submittedDate;
    private final int score;
    private final Map<String, Integer> answers;
    public Result(String resultId, String studentId, String testId,
                  int score, Map<String, Integer> answers, String submittedDate) {
        this.resultId = resultId; this.studentId = studentId; this.testId = testId;
        this.score = score; this.answers = answers; this.submittedDate = submittedDate;
    }
    public String              getResultId()     { return resultId;     }
    public String              getStudentId()    { return studentId;    }
    public String              getTestId()       { return testId;       }
    public int                 getScore()        { return score;        }
    public Map<String,Integer> getAnswers()      { return answers;      }
    public String              getSubmittedDate(){ return submittedDate; }
}
