package com.recsys;

import com.sun.net.httpserver.*;
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.charset.StandardCharsets;
import java.nio.file.*;
import java.security.MessageDigest;
import java.util.*;
import java.util.regex.*;
import java.util.stream.Collectors;

/**
 * Web layer: a tiny HTTP server (JDK only) exposing the REST API used by web/index.html.
 * Roles: ADMIN (settings, effectiveness analysis, user management) and USER (rate, preferences, recommendations).
 * Storage: MySQL through JDBC (JdbcStore) when db.properties is valid, otherwise in memory (MemoryStore).
 */
public class App {
    record User(int id, String name, String email, String password, String role, Set<String> prefs) {}
    record Item(int id, String name, String category) {}
    static class ApiError extends RuntimeException { final int code; ApiError(int c, String m) { super(m); code = c; } }

    static final String[] CATEGORIES = {"Sci-Fi", "Action", "Comedy", "Bollywood", "Drama"};
    static final RecommendationEngine engine = new RecommendationEngine();
    static final Map<Integer, User> users = new LinkedHashMap<>();
    static final Map<Integer, Item> items = new LinkedHashMap<>();
    static final Map<String, Integer> sessions = new HashMap<>();   // token -> userId
    static int nextUserId = 1, shown = 0, clicked = 0;               // recommendation log counters
    static int k = 3, minCommon = 2; static double minSim = 0.1;     // admin settings
    static DataStore store = new MemoryStore();                      // MySQL (JdbcStore) when db.properties works

    // ---------------------------------------------------------------- startup
    public static void main(String[] args) throws Exception {
        store = JdbcStore.fromConfig("db.properties");
        if (!store.load()) { seed(); store.saveAll(); }   // empty or no database: create demo data (and save it)
        applySettings();
        System.out.println("Storage: " + store.name());
        int port = args.length > 0 ? Integer.parseInt(args[0]) : 8080;
        HttpServer server = HttpServer.create(new InetSocketAddress(port), 0);
        server.createContext("/", App::handle);
        server.start();
        System.out.println("Running at http://localhost:" + port + "   (admin@demo.com / admin123, aman@demo.com / user123)");
    }

    /** Pushes the admin settings into the recommendation engine. */
    static void applySettings() { engine.setNeighbourCount(k); engine.setMinCommonItems(minCommon); engine.setMinSimilarity(minSim); }

    /** SHA-256 hex of a password (passwords are never stored in plain text). */
    static String hash(String pw) {
        try {
            StringBuilder sb = new StringBuilder();
            for (byte b : MessageDigest.getInstance("SHA-256").digest(pw.getBytes(StandardCharsets.UTF_8))) sb.append(String.format("%02x", b));
            return sb.toString();
        } catch (java.security.NoSuchAlgorithmException e) { throw new IllegalStateException(e); }
    }

    static User addUser(String name, String email, String pw, String role) {
        User u = new User(nextUserId++, name, email, hash(pw), role, new HashSet<>());
        users.put(u.id(), u); return u;
    }

    static void seed() {
        addUser("Admin", "admin@demo.com", "admin123", "ADMIN");
        String[][] list = {{"Inception","Sci-Fi"},{"Interstellar","Sci-Fi"},{"The Matrix","Sci-Fi"},{"Dune","Sci-Fi"},
            {"The Dark Knight","Action"},{"Mad Max: Fury Road","Action"},{"John Wick","Action"},{"Avengers: Endgame","Action"},
            {"3 Idiots","Comedy"},{"Hera Pheri","Comedy"},{"Chhichhore","Comedy"},{"Dangal","Bollywood"},
            {"Zindagi Na Milegi Dobara","Bollywood"},{"Taare Zameen Par","Drama"},{"Swades","Drama"},{"Queen","Drama"}};
        for (int i = 0; i < list.length; i++) {
            items.put(i + 1, new Item(i + 1, list[i][0], list[i][1]));
            engine.addItem(i + 1, list[i][0]);
        }
        String[] names = {"Aman","Riya","Karan","Sneha","Rohit","Priya","Vikram","Neha"};
        Random rnd = new Random(42);
        for (int i = 0; i < names.length; i++) {
            User u = addUser(names[i], names[i].toLowerCase() + "@demo.com", "user123", "USER");
            boolean actionFan = i % 2 == 0;   // even users like Sci-Fi/Action, odd users like the rest
            for (Item t : items.values()) {
                if (rnd.nextDouble() > 0.65) continue;   // leave some items unrated
                boolean actionItem = t.category().equals("Sci-Fi") || t.category().equals("Action");
                engine.addRating(u.id(), t.id(), actionFan == actionItem ? 4 + rnd.nextInt(2) : 1 + rnd.nextInt(2));
            }
        }
    }

    // ---------------------------------------------------------------- HTTP plumbing
    static void handle(HttpExchange ex) throws IOException {
        int code = 200; String type = "application/json"; byte[] out;
        try {
            String path = ex.getRequestURI().getPath();
            if (!path.startsWith("/api/")) {
                type = "text/html; charset=utf-8"; out = Files.readAllBytes(findPage());
            } else {
                Map<String, String> p = parse(new String(ex.getRequestBody().readAllBytes(), StandardCharsets.UTF_8));
                String res; synchronized (App.class) { res = route(ex, path, p); }
                out = res.getBytes(StandardCharsets.UTF_8);
            }
        } catch (ApiError e) { code = e.code; out = j(Map.of("error", e.getMessage())).getBytes(StandardCharsets.UTF_8);
        } catch (NumberFormatException e) { code = 400; out = j(Map.of("error", "Invalid number")).getBytes(StandardCharsets.UTF_8);
        } catch (Exception e) { code = 500; out = j(Map.of("error", "Server error")).getBytes(StandardCharsets.UTF_8); e.printStackTrace(); }
        ex.getResponseHeaders().set("Content-Type", type);
        ex.sendResponseHeaders(code, out.length);
        ex.getResponseBody().write(out); ex.close();
    }

    /** Finds web/index.html whether the app is started from the project root, src/ or out/ (or an IDE). */
    static Path findPage() throws IOException {
        for (String f : new String[]{"web/index.html", "../web/index.html", "../../web/index.html", "../../../web/index.html"})
            if (Files.exists(Path.of(f))) return Path.of(f);
        throw new IOException("web/index.html not found - run the app from the project folder");
    }

    /** Reads flat JSON like {"a":"x","b":5} into a map (enough for our simple requests). */
    static Map<String, String> parse(String body) {
        Map<String, String> m = new HashMap<>();
        Matcher mt = Pattern.compile("\"(\\w+)\"\\s*:\\s*(?:\"((?:[^\"\\\\]|\\\\.)*)\"|([^,}\\s]+))").matcher(body);
        while (mt.find()) m.put(mt.group(1), mt.group(2) != null ? mt.group(2) : mt.group(3));
        return m;
    }

    /** Minimal JSON writer for String / Number / Boolean / Map / Collection. */
    static String j(Object o) {
        if (o instanceof String s) return "\"" + s.replace("\\", "\\\\").replace("\"", "\\\"") + "\"";
        if (o instanceof Double d) return String.format(Locale.US, "%.2f", d);
        if (o instanceof Map<?, ?> m) return m.entrySet().stream().map(e -> j(e.getKey().toString()) + ":" + j(e.getValue())).collect(Collectors.joining(",", "{", "}"));
        if (o instanceof Collection<?> c) return c.stream().map(App::j).collect(Collectors.joining(",", "[", "]"));
        return String.valueOf(o);
    }

    // ---------------------------------------------------------------- API routes
    static String route(HttpExchange ex, String path, Map<String, String> p) {
        if (path.equals("/api/login")) return login(p.get("email"), p.get("password"));
        if (path.equals("/api/register")) {
            String name = p.getOrDefault("name", "").trim(), email = p.getOrDefault("email", "").trim().toLowerCase(), pw = p.getOrDefault("password", "");
            if (name.isEmpty() || !email.contains("@") || pw.length() < 4) throw new ApiError(400, "Enter name, valid email and password (min 4 chars)");
            if (users.values().stream().anyMatch(u -> u.email().equals(email))) throw new ApiError(409, "Email already registered");
            store.saveUser(addUser(name, email, pw, "USER"));
            return login(email, pw);
        }
        Integer uid = sessions.get(ex.getRequestHeaders().getFirst("X-Token"));
        User u = uid == null ? null : users.get(uid);
        if (u == null) throw new ApiError(401, "Please login");
        if (path.startsWith("/api/admin/") && !u.role().equals("ADMIN")) throw new ApiError(403, "Admin only");

        Map<Integer, Map<Integer, Double>> all = engine.allRatings();
        switch (path) {
            case "/api/items": {   // all items with my rating + average
                List<Map<String, Object>> out = new ArrayList<>();
                for (Item t : items.values()) {
                    double sum = 0; int cnt = 0;
                    for (Map<Integer, Double> r : all.values()) if (r.containsKey(t.id())) { sum += r.get(t.id()); cnt++; }
                    out.add(Map.of("id", t.id(), "name", t.name(), "category", t.category(),
                            "mine", all.getOrDefault(u.id(), Map.of()).getOrDefault(t.id(), 0.0).intValue(), "avg", cnt == 0 ? 0.0 : sum / cnt));
                }
                return j(out);
            }
            case "/api/rate": {
                int item = Integer.parseInt(p.get("itemId")), rating = Integer.parseInt(p.get("rating"));
                if (!items.containsKey(item) || rating < 1 || rating > 5) throw new ApiError(400, "Invalid rating");
                engine.addRating(u.id(), item, rating);
                store.saveRating(u.id(), item, rating);
                return j(Map.of("message", "Rating saved"));
            }
            case "/api/recommendations": {   // collaborative filtering + small boost for preferred categories
                List<Map<String, Object>> out = new ArrayList<>();
                for (Recommendation r : engine.recommend(u.id(), 12)) {
                    Item it = items.get(r.itemId());
                    if (it == null || r.predictedScore() < 3.5 || all.getOrDefault(u.id(), Map.of()).containsKey(it.id())) continue;   // only show items predicted to be liked
                    boolean hit = u.prefs().contains(it.category());
                    out.add(new HashMap<>(Map.of("id", it.id(), "name", it.name(), "category", it.category(),
                            "score", Math.min(5.0, r.predictedScore() + (hit ? 0.5 : 0)),
                            "reason", r.reason() + (hit ? " + matches your preferences" : ""))));
                }
                out.sort((a, b) -> Double.compare((Double) b.get("score"), (Double) a.get("score")));
                List<Map<String, Object>> top = out.size() > 6 ? out.subList(0, 6) : out;
                shown += top.size();
                for (Map<String, Object> t : top) store.logShown(u.id(), (Integer) t.get("id"), (Double) t.get("score"));
                return j(top);
            }
            case "/api/click": clicked++; store.logClick(u.id(), Integer.parseInt(p.getOrDefault("itemId", "0"))); return j(Map.of("message", "Logged"));
            case "/api/preferences": {
                if (p.containsKey("prefs")) {
                    u.prefs().clear();
                    for (String s : p.get("prefs").split(",")) if (Arrays.asList(CATEGORIES).contains(s.trim())) u.prefs().add(s.trim());
                    store.savePrefs(u.id(), u.prefs());
                    return j(Map.of("message", "Preferences updated successfully"));
                }
                return j(Map.of("prefs", u.prefs(), "all", Arrays.asList(CATEGORIES)));
            }
            case "/api/admin/settings": {
                if (p.containsKey("k")) {
                    int nk = Integer.parseInt(p.get("k")), nc = Integer.parseInt(p.get("minCommon")); double ns = Double.parseDouble(p.get("minSim"));
                    if (nk < 1 || nk > 20 || nc < 1 || nc > 10 || ns < 0 || ns > 1) throw new ApiError(400, "Values out of range");
                    k = nk; minCommon = nc; minSim = ns;
                    applySettings();
                    store.saveSetting("neighbour_count", String.valueOf(k)); store.saveSetting("min_similarity", String.valueOf(minSim)); store.saveSetting("min_common_items", String.valueOf(minCommon));
                    return j(Map.of("message", "Settings updated successfully"));
                }
                return j(Map.of("k", k, "minSim", minSim, "minCommon", minCommon));
            }
            case "/api/admin/analysis": return analysis(all);
            case "/api/admin/users": {
                List<Map<String, Object>> out = new ArrayList<>();
                for (User x : users.values()) out.add(Map.of("id", x.id(), "name", x.name(), "email", x.email(), "role", x.role(),
                        "ratings", all.getOrDefault(x.id(), Map.of()).size(), "prefs", x.prefs()));
                return j(out);
            }
            case "/api/admin/deleteUser": {
                int id = Integer.parseInt(p.get("id")); User t = users.get(id);
                if (t == null || t.role().equals("ADMIN")) throw new ApiError(400, "Cannot delete this user");
                users.remove(id); engine.removeUser(id); store.deleteUser(id); sessions.values().removeIf(v -> v == id);
                return j(Map.of("message", "User deleted"));
            }
            default: throw new ApiError(404, "Unknown endpoint");
        }
    }

    static String login(String email, String pw) {
        if (email == null || pw == null) throw new ApiError(400, "Email and password required");
        User u = users.values().stream().filter(x -> x.email().equalsIgnoreCase(email.trim()) && x.password().equals(hash(pw))).findFirst()
                .orElseThrow(() -> new ApiError(401, "Wrong email or password"));
        String token = UUID.randomUUID().toString(); sessions.put(token, u.id());
        return j(Map.of("token", token, "role", u.role(), "name", u.name()));
    }

    /** Admin "Recommendation Effectiveness Analysis": accuracy (MAE), click-through rate, charts data, suggestions. */
    static String analysis(Map<Integer, Map<Integer, Double>> all) {
        double mae = engine.evaluateMAE(); if (Double.isNaN(mae)) mae = 0;
        double ctr = shown == 0 ? 0 : clicked * 100.0 / shown;
        int total = all.values().stream().mapToInt(Map::size).sum();
        Map<String, Integer> byCat = new TreeMap<>(); Map<Integer, List<Double>> byItem = new HashMap<>();
        all.values().forEach(r -> r.forEach((i, v) -> { byCat.merge(items.get(i).category(), 1, Integer::sum); byItem.computeIfAbsent(i, x -> new ArrayList<>()).add(v); }));
        List<Map<String, Object>> top = byItem.entrySet().stream()
                .map(e -> Map.<String, Object>of("name", items.get(e.getKey()).name(), "avg", e.getValue().stream().mapToDouble(d -> d).average().orElse(0), "count", e.getValue().size()))
                .sorted((a, b) -> Double.compare((Double) b.get("avg"), (Double) a.get("avg"))).limit(5).collect(Collectors.toList());
        List<String> tips = new ArrayList<>();
        if (mae > 1.0) tips.add("Prediction error is high (MAE " + String.format(Locale.US, "%.2f", mae) + "): try lowering the minimum similarity or raising the neighbour count.");
        else tips.add("Prediction error is healthy (MAE " + String.format(Locale.US, "%.2f", mae) + ").");
        if (shown > 0 && ctr < 15) tips.add("Click-through rate is low: encourage users to set preferences so suggestions are better targeted.");
        if (total < users.size() * 5) tips.add("Ratings are sparse: more user ratings will improve recommendation quality.");
        return j(Map.of("users", users.size() - 1, "ratings", total, "mae", mae, "ctr", ctr, "shown", shown, "clicked", clicked,
                "storage", store.name(), "byCategory", byCat, "topItems", top, "suggestions", tips));
    }
}
