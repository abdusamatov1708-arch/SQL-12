-- =========================================================================
-- 1. ESKI JADVALLARNI TOZALASH (KETMA-KETLIKDA)
-- =========================================================================
DROP TABLE IF EXISTS post_tags CASCADE;
DROP TABLE IF EXISTS comments CASCADE;
DROP TABLE IF EXISTS tags CASCADE;
DROP TABLE IF EXISTS posts CASCADE;
DROP TABLE IF EXISTS users CASCADE;

-- =========================================================================
-- 2. JADVALLAR SXEMASINI YARATISH (SCHEMA)
-- =========================================================================

-- Foydalanuvchilar jadvali
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Postlar jadvali (ON DELETE RESTRICT bilan)
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id) ON DELETE RESTRICT, 
    title VARCHAR(255) NOT NULL,
    body TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Izohlar jadvali (ON DELETE CASCADE bilan)
CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    post_id INT REFERENCES posts(id) ON DELETE CASCADE,  
    user_id INT REFERENCES users(id) ON DELETE RESTRICT,
    comment_text TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Teglar jadvali
CREATE TABLE tags (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL
);

-- Post va Teglarni bog'lovchi jadval (M:N munosabat)
CREATE TABLE post_tags (
    post_id INT REFERENCES posts(id) ON DELETE CASCADE,
    tag_id INT REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);

-- =========================================================================
-- 3. FOREIGN KEY USTUNLARIGA INDEKSLAR QO'SHISH (5 TA INDEKS)
-- =========================================================================
CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_comments_post_id ON comments(post_id);
CREATE INDEX idx_comments_user_id ON comments(user_id);
CREATE INDEX idx_post_tags_post_id ON post_tags(post_id);
CREATE INDEX idx_post_tags_tag_id ON post_tags(tag_id);


-- =========================================================================
-- 4. TRANZAKSIYA ICHIDA TEST MA'LUMOTLARINI GENERATSIYA QILISH
-- =========================================================================
BEGIN;

-- A) 1 000 ta foydalanuvchi
INSERT INTO users (username)
SELECT 'user_' || i FROM generate_series(1, 1000) AS i;

-- B) 5 000 ta post (tasodifiy foydalanuvchilarga biriktirilgan)
INSERT INTO posts (user_id, title, body)
SELECT 
    (random() * 999 + 1)::INT,
    'Post sarlavhasi ' || i,
    'Bu yerda post matni joylashgan. Qator raqami: ' || i
FROM generate_series(1, 5000) AS i;

-- D) 50 ta teg
INSERT INTO tags (name)
SELECT 'tag_' || i FROM generate_series(1, 50) AS i;

-- E) 15 000 ta izoh (faqat dastlabki 4500 ta postga tasodifiy taqsimlanadi)
INSERT INTO comments (post_id, user_id, comment_text)
SELECT 
    (random() * 4499 + 1)::INT, 
    (random() * 999 + 1)::INT,
    'Ajoyib post ekan! Tartib raqami: ' || i
FROM generate_series(1, 15000) AS i;

-- F) 12 000 ta Post-Teg bog'liqligi (M:N munosabat uchun)
INSERT INTO post_tags (post_id, tag_id)
SELECT DISTINCT
    (random() * 4999 + 1)::INT,
    (random() * 49 + 1)::INT
FROM generate_series(1, 12000) AS i
ON CONFLICT DO NOTHING;

COMMIT;


-- =========================================================================
-- 5. ANALITIK HISOBOTLAR
-- =========================================================================

-- 1-Hisobot: Eng faol foydalanuvchilar (Top 5)
SELECT u.id, u.username, COUNT(p.id) AS posts_count
FROM users u
JOIN posts p ON u.id = p.user_id
GROUP BY u.id, u.username
ORDER BY posts_count DESC
LIMIT 5;

-- 2-Hisobot: Eng mashhur teglar (Top 5)
SELECT t.name, COUNT(pt.post_id) AS post_count
FROM tags t
JOIN post_tags pt ON t.id = pt.tag_id
GROUP BY t.id, t.name
ORDER BY post_count DESC
LIMIT 5;

-- 3-Hisobot: Har bir postning izohlari soni
SELECT p.id, p.title, COUNT(c.id) AS comments_count
FROM posts p
LEFT JOIN comments c ON p.id = c.post_id
GROUP BY p.id, p.title
ORDER BY comments_count DESC;

-- 4-Hisobot: Umuman izoh yozilmagan (0 izohli) postlar
SELECT p.id, p.title
FROM posts p
LEFT JOIN comments c ON p.id = c.post_id
WHERE c.id IS NULL;


-- =========================================================================
-- 6. EXPLAIN ANALYZE BILAN OPTIMIZATSIYANI TEKSHIRISH
-- =========================================================================

-- 4-hisobot so'rovining samaradorligini tekshiramiz.
-- Bu yerda biz yaratgan `idx_comments_post_id` indeksidan foydalaniladi.
EXPLAIN ANALYZE 
SELECT p.id, p.title
FROM posts p
LEFT JOIN comments c ON p.id = c.post_id
WHERE c.id IS NULL;
