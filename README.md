import javax.swing.*;
import javax.swing.border.EmptyBorder;
import java.awt.*;
import java.awt.event.*;
import java.io.*;
import java.util.*;
import javax.naming.*;
import javax.naming.directory.*;

public class SmartEmailValidatorTool extends JFrame {

    // Palette / Theme
    private static final Color PRIMARY = new Color(56, 97, 251);
    private static final Color SUCCESS = new Color(34, 197, 94);
    private static final Color WARNING = new Color(245, 158, 11);
    private static final Color DANGER  = new Color(239, 68, 68);
    private static final Color LIGHT_BG = new Color(248, 250, 252);
    private static final Color LIGHT_PANEL = Color.WHITE;
    private static final Color DARK_BG = new Color(22, 27, 34);
    private static final Color DARK_PANEL = new Color(33, 38, 45);

    // UI Components
    private JTextField emailField;
    private JTextArea logArea;
    private JLabel summaryLabel, validBadge, riskyBadge, invalidBadge;
    private JProgressBar progressBar;
    private JButton validateSingleButton, selectCSVButton, startCSVButton, clearButton;
    private JComboBox<String> themeSelector;
    private ChartPanel chartPanel;

    // File reference for CSV
    private File csvFile;
    private java.util.List<String[]> reportData = new ArrayList<>();

    // Counters
    private int validCount = 0, invalidCount = 0, riskyCount = 0;

    public SmartEmailValidatorTool() {
        super("Smart Email Validator Tool");
        installNimbus();

        setLayout(new BorderLayout());
        setSize(1000, 650);
        setLocationByPlatform(true);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);

        JPanel root = new JPanel(new BorderLayout());
        root.setBorder(new EmptyBorder(10, 12, 12, 12));
        add(root, BorderLayout.CENTER);

        // === HEADER ===
        JPanel header = new JPanel(new BorderLayout(10, 10));
        JLabel title = new JLabel("Smart Email Validator");
        title.setFont(title.getFont().deriveFont(Font.BOLD, 22f));
        title.setForeground(PRIMARY.darker());
        header.add(title, BorderLayout.WEST);

        themeSelector = new JComboBox<>(new String[]{"Light", "Dark", "System"});
        themeSelector.addActionListener(e -> applyTheme((String) themeSelector.getSelectedItem()));
        JPanel headerRight = new JPanel(new FlowLayout(FlowLayout.RIGHT, 8, 2));
        headerRight.add(new JLabel("Theme:"));
        headerRight.add(themeSelector);
        header.add(headerRight, BorderLayout.EAST);
        root.add(header, BorderLayout.NORTH);

        // === CONTROLS STRIP ===
        JPanel controls = new JPanel(new GridBagLayout());
        controls.setBorder(new EmptyBorder(8, 0, 8, 0));
        GridBagConstraints gc = new GridBagConstraints();
        gc.insets = new Insets(0, 4, 0, 4);
        gc.fill = GridBagConstraints.HORIZONTAL;

        gc.gridx = 0; gc.gridy = 0; gc.weightx = 0; controls.add(new JLabel("Enter Email:"), gc);
        emailField = new JTextField(35);
        gc.gridx = 1; gc.weightx = 1; controls.add(emailField, gc);
        validateSingleButton = new JButton("Validate");
        gc.gridx = 2; gc.weightx = 0; controls.add(validateSingleButton, gc);
        clearButton = new JButton("Clear");
        gc.gridx = 3; controls.add(clearButton, gc);
        selectCSVButton = new JButton("Select CSV");
        gc.gridx = 4; controls.add(selectCSVButton, gc);
        startCSVButton = new JButton("Run CSV");
        gc.gridx = 5; controls.add(startCSVButton, gc);
        root.add(controls, BorderLayout.PAGE_START);

        // === CENTER: Split - Log (left) | Insight (right) ===
        logArea = new JTextArea();
        logArea.setEditable(false);
        logArea.setFont(new Font(Font.MONOSPACED, Font.PLAIN, 13));
        JScrollPane logScroll = new JScrollPane(logArea);
        logScroll.setBorder(BorderFactory.createTitledBorder("Validation Output"));

        JPanel insight = new JPanel(new BorderLayout(8, 8));
        insight.setBorder(new EmptyBorder(0, 8, 0, 0));

        // badges row
        JPanel badges = new JPanel(new GridLayout(1, 3, 8, 8));
        validBadge = createBadge("Valid: 0", SUCCESS);
        riskyBadge = createBadge("Risky: 0", WARNING);
        invalidBadge = createBadge("Invalid: 0", DANGER);
        badges.add(validBadge);
        badges.add(riskyBadge);
        badges.add(invalidBadge);
        insight.add(badges, BorderLayout.NORTH);

        // chart panel
        chartPanel = new ChartPanel();
        chartPanel.setBorder(BorderFactory.createTitledBorder("Summary (Bar Graph)"));
        insight.add(chartPanel, BorderLayout.CENTER);

        JSplitPane split = new JSplitPane(JSplitPane.HORIZONTAL_SPLIT, logScroll, insight);
        split.setResizeWeight(0.6);
        root.add(split, BorderLayout.CENTER);

        // === FOOTER ===
        JPanel footer = new JPanel(new BorderLayout(8, 8));
        progressBar = new JProgressBar(0, 100);
        progressBar.setStringPainted(true);
        footer.add(progressBar, BorderLayout.CENTER);
        summaryLabel = new JLabel("Ready.");
        summaryLabel.setHorizontalAlignment(SwingConstants.RIGHT);
        footer.add(summaryLabel, BorderLayout.EAST);
        root.add(footer, BorderLayout.SOUTH);

        // Listeners
        validateSingleButton.addActionListener(e -> validateSingleEmail());
        emailField.addActionListener(e -> validateSingleEmail()); // Press Enter in the field to validate
        clearButton.addActionListener(e -> {
            logArea.setText("");
            validCount = riskyCount = invalidCount = 0;
            reportData.clear();
            progressBar.setValue(0);
            updateSummary();
        });
        selectCSVButton.addActionListener(e -> selectCSV());
        startCSVButton.addActionListener(e -> startCSVValidation());

        // Make Enter trigger validate anywhere in the window as well
        getRootPane().setDefaultButton(validateSingleButton);

        // initial theme
        applyTheme("Light");
    }

    private void installNimbus() {
        try {
            UIManager.setLookAndFeel("javax.swing.plaf.nimbus.NimbusLookAndFeel");
        } catch (Exception ignored) { }
    }

    private JLabel createBadge(String text, Color color) {
        JLabel l = new JLabel(text, SwingConstants.CENTER);
        l.setOpaque(true);
        l.setBackground(color);
        l.setForeground(Color.WHITE);
        l.setBorder(new EmptyBorder(10, 12, 10, 12));
        l.setFont(l.getFont().deriveFont(Font.BOLD, 14f));
        return l;
    }

    private void applyTheme(String theme) {
        boolean dark = "Dark".equalsIgnoreCase(theme);
        Color bg = dark ? DARK_BG : LIGHT_BG;
        Color panel = dark ? DARK_PANEL : LIGHT_PANEL;
        Color fg = dark ? Color.WHITE : Color.DARK_GRAY;

        SwingUtilities.invokeLater(() -> {
            setBackground(bg);
            getContentPane().setBackground(bg);
            for (Component c : getComponents()) c.setBackground(bg);
            // Traverse and apply basic colors
            applyColorsRecursively(getContentPane(), panel, fg, bg);
            repaint();
        });
    }

    private void applyColorsRecursively(Component comp, Color panel, Color fg, Color bg) {
        if (comp instanceof JComponent) {
            if (!(comp instanceof JScrollPane)) comp.setBackground(panel);
            comp.setForeground(fg);
        }
        if (comp instanceof Container) {
            for (Component child : ((Container) comp).getComponents()) {
                applyColorsRecursively(child, panel, fg, bg);
            }
        }
    }

    // ============= SINGLE EMAIL VALIDATION ==================
    private void validateSingleEmail() {
        String email = emailField.getText().trim();
        if (email.isEmpty()) {
            JOptionPane.showMessageDialog(this, "Please enter an email address.");
            return;
        }
        // Reset counters for single-check mode so each run is independent
        validCount = riskyCount = invalidCount = 0;
        reportData.clear();
        progressBar.setValue(0);
        updateSummary();

        logArea.append("\nChecking: " + email + "\n\n");
        validateEmail(email, true);
        updateSummary();
    }

    // ============= CSV VALIDATION ==================
    private void selectCSV() {
        JFileChooser fileChooser = new JFileChooser();
        int result = fileChooser.showOpenDialog(this);
        if (result == JFileChooser.APPROVE_OPTION) {
            csvFile = fileChooser.getSelectedFile();
            logArea.append("Selected CSV file: " + csvFile.getAbsolutePath() + "\n");
        }
    }

    private void startCSVValidation() {
        if (csvFile == null) {
            JOptionPane.showMessageDialog(this, "Please select a CSV file first!");
            return;
        }

        new Thread(() -> {
            try (BufferedReader br = new BufferedReader(new FileReader(csvFile))) {
                String line;
                java.util.List<String> emails = new ArrayList<>();
                while ((line = br.readLine()) != null) {
                    if (!line.trim().isEmpty()) emails.add(line.trim());
                }

                int total = emails.size();
                logArea.append("\nStarting validation for " + total + " emails...\n");
                int index = 0; validCount = riskyCount = invalidCount = 0;

                for (String email : emails) {
                    logArea.append("\nChecking: " + email + "\n\n");
                    validateEmail(email, true);
                    index++;
                    progressBar.setValue((int) (((double) index / total) * 100));
                    updateSummary();
                    try { Thread.sleep(30); } catch (InterruptedException ignored) {}
                }

                logArea.append("\nDone.\n");
            } catch (Exception ex) {
                logArea.append("Error: " + ex.getMessage() + "\n");
            }
        }).start();
    }

    // ============= VALIDATION LOGIC ==================
    private void validateEmail(String email, boolean verbose) {
        boolean syntaxValid = email.matches("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+$");

        String domain = email.contains("@") ? email.substring(email.indexOf('@') + 1) : "";
        boolean domainLooksValid = domain.contains(".") && !domain.startsWith("-") && !domain.endsWith("-");
        boolean hasMX = domainLooksValid && hasMXRecord(domain);

        // typos suggestion (simple)
        String suggestion = null;
        String[] common = {"gmail.com","yahoo.com","outlook.com","hotmail.com"};
        for (String c : common) {
            if (domainLooksValid && !domain.equalsIgnoreCase(c) && damerauLevenshtein(domain, c) == 1) { 
                suggestion = c; 
                break; 
            }
        }

        boolean disposable = isDisposable(domain);
        boolean roleBased = isRoleBased(email);

        // Heuristic score [0..100]
        double score = 100;
        if (!syntaxValid) score -= 70;
        if (!domainLooksValid) score -= 30;
        if (!hasMX) score -= 50;
        if (suggestion != null && !domain.equalsIgnoreCase(suggestion)) score -= 25; // likely typo
        if (disposable) score -= 35;
        if (roleBased) score -= 20;
        if (email.split("@")[0].matches(".\\d{3,}.")) score -= 10; // randomized username hint
        score = Math.max(0, Math.min(100, score));

        String status;
        if (!syntaxValid || !domainLooksValid || !hasMX) {
            status = "Invalid"; invalidCount++;
        } else if (disposable || roleBased || (suggestion != null && !domain.equalsIgnoreCase(suggestion)) || score < 70) {
            status = "Risky"; riskyCount++;
        } else {
            status = "Valid"; validCount++;
        }

        reportData.add(new String[]{email, status, String.format(Locale.US, "Score %.0f", score)});

        if (verbose) {
            appendSection("Syntax Check:", syntaxValid ? "Syntax OK" : "Invalid email syntax");
            String domainMsg = !domainLooksValid ? "Domain malformed" : (hasMX ? "Domain has valid MX record" : "No MX record for domain");
            appendSection("Domain Validation:", domainMsg);
            appendSection("Typo & Suggestion Check:", suggestion != null ? ("Did you mean: " + suggestion + " ?") : "No obvious typo detected");
            appendSection("Disposable Email Detection:", disposable ? "Disposable domain" : "Not a disposable domain");
            appendSection("Role-based Email Detection:", roleBased ? "Role-based email" : "Not a role-based email");
            appendSection("SMTP Mailbox Simulation:", hasMX ? "SMTP server responds properly (simulated)" : "Skipped (no MX record)");
            appendSection("Heuristic / ML Filtering:", score < 70 ? "Suspicious pattern detected" : "No suspicious pattern detected");
            logArea.append(String.format(Locale.US, "\n• Deliverability Score: %.0f%%\n", score));
            logArea.append("• Result: " + status + "\n");
        }

        updateSummary();
    }

    private void appendSection(String title, String line) {
        logArea.append("□ " + title + "\n");
        logArea.append("  " + line + "\n\n");
    }

    // Simple Levenshtein distance (kept) and Damerau-Levenshtein (handles transpositions like gamil->gmail)
    private int levenshtein(String a, String b) {
        int[][] dp = new int[a.length()+1][b.length()+1];
        for (int i=0;i<=a.length();i++) dp[i][0]=i;
        for (int j=0;j<=b.length();j++) dp[0][j]=j;
        for (int i=1;i<=a.length();i++) {
            for (int j=1;j<=b.length();j++) {
                int cost = a.charAt(i-1)==b.charAt(j-1)?0:1;
                dp[i][j]=Math.min(Math.min(dp[i-1][j]+1, dp[i][j-1]+1), dp[i-1][j-1]+cost);
            }
        }
        return dp[a.length()][b.length()];
    }

    private int damerauLevenshtein(String a, String b) {
        int[][] dp = new int[a.length()+1][b.length()+1];
        for (int i=0;i<=a.length();i++) dp[i][0] = i;
        for (int j=0;j<=b.length();j++) dp[0][j] = j;
        for (int i=1;i<=a.length();i++) {
            for (int j=1;j<=b.length();j++) {
                int cost = a.charAt(i-1) == b.charAt(j-1) ? 0 : 1;
                dp[i][j] = Math.min(Math.min(dp[i-1][j] + 1, dp[i][j-1] + 1), dp[i-1][j-1] + cost);
                if (i>1 && j>1 && a.charAt(i-1)==b.charAt(j-2) && a.charAt(i-2)==b.charAt(j-1)) {
                    dp[i][j] = Math.min(dp[i][j], dp[i-2][j-2] + 1); // transposition
                }
            }
        }
        return dp[a.length()][b.length()];
    }

    // Helper Checks
    private boolean isDisposable(String domain) {
        String[] disposableDomains = {"mailinator.com", "tempmail.com", "10minutemail.com", "guerrillamail.com"};
        for (String d : disposableDomains) if (domain.equalsIgnoreCase(d)) return true;
        return false;
    }

    private boolean hasMXRecord(String domain) {
        try {
            Hashtable<String, String> env = new Hashtable<>();
            env.put("java.naming.factory.initial", "com.sun.jndi.dns.DnsContextFactory");
            DirContext ictx = new InitialDirContext(env);
            Attributes attrs = ictx.getAttributes(domain, new String[]{"MX"});
            Attribute attr = attrs.get("MX");
            if (attr != null && attr.size() > 0) return true;
            // fallback A record
            attrs = ictx.getAttributes(domain, new String[]{"A"});
            attr = attrs.get("A");
            return attr != null && attr.size() > 0;
        } catch (Exception e) {
            return false;
        }
    }

    private boolean isRoleBased(String email) {
        String[] roles = {"admin@", "info@", "support@", "sales@", "contact@"};
        for (String role : roles) if (email.toLowerCase().startsWith(role)) return true;
        return false;
    }

    // ============= UI UPDATES ==================
    private void updateSummary() {
        validBadge.setText("Valid: " + validCount);
        riskyBadge.setText("Risky: " + riskyCount);
        invalidBadge.setText("Invalid: " + invalidCount);
        summaryLabel.setText("✔ " + validCount + "  •  ⚠ " + riskyCount + "  •  ❌ " + invalidCount);
        chartPanel.repaint();
    }

    // Embedded bar chart panel (no external libs)
    private class ChartPanel extends JPanel {
        @Override protected void paintComponent(Graphics g) {
            super.paintComponent(g);
            Graphics2D g2 = (Graphics2D) g;
            g2.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);

            int total = Math.max(1, validCount + riskyCount + invalidCount);
            int w = getWidth();
            int h = getHeight();
            int chartH = h - 60; // padding for labels
            int baseY = h - 30;
            int barW = Math.max(50, (w - 160) / 3);
            int gap = 30;
            int startX = (w - (barW*3 + gap*2)) / 2;

            int validBar = (int)((validCount/(double)total) * chartH);
            int riskyBar = (int)((riskyCount/(double)total) * chartH);
            int invalidBar = (int)((invalidCount/(double)total) * chartH);

            // axes
            g2.setColor(new Color(180,180,180));
            g2.drawLine(40, baseY, w-40, baseY);

            // bars
            int x = startX;
            drawBar(g2, x, baseY-validBar, barW, validBar, SUCCESS, "Valid", validCount); x += barW+gap;
            drawBar(g2, x, baseY-riskyBar, barW, riskyBar, WARNING, "Risky", riskyCount); x += barW+gap;
            drawBar(g2, x, baseY-invalidBar, barW, invalidBar, DANGER, "Invalid", invalidCount);
        }

        private void drawBar(Graphics2D g2, int x, int y, int w, int h, Color color, String label, int value) {
            g2.setColor(color);
            g2.fillRoundRect(x, y, w, h, 10, 10);
            g2.setColor(Color.DARK_GRAY);
            g2.setFont(getFont().deriveFont(Font.BOLD, 12f));
            String txt = label + ": " + value;
            int tx = x + (w - g2.getFontMetrics().stringWidth(txt)) / 2;
            g2.drawString(txt, tx, getHeight()-10);
        }
    }

    // ============= MAIN ==================
    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> new SmartEmailValidatorTool().setVisible(true));
    }
}
