#!/usr/bin/env python3
"""
TrendRadar RSS 标题去重补丁
功能：对当日 SQLite 数据库中的 rss_items 表按标题相似度去重，
      保留最早出现或排名最高的一条，删除/忽略其余重复项。
用法：python scripts/rss_title_dedup.py [--db path/to/YYYY-MM-DD.db] [--threshold 0.9]
      不指定 --db 时自动查找 output/news/ 下最新的 .db 文件。
"""

import argparse
import sqlite3
import sys
from difflib import SequenceMatcher
from pathlib import Path
from datetime import datetime

SIMILARITY_THRESHOLD = 0.90  # 默认阈值，可调


def find_latest_db(base_dir: Path) -> Path | None:
    """在 output/news/ 下查找最新的 .db 文件"""
    news_dir = base_dir / "output" / "news"
    if not news_dir.exists():
        return None
    dbs = sorted(news_dir.glob("*.db"), key=lambda p: p.stat().st_mtime, reverse=True)
    return dbs[0] if dbs else None


def normalize_title(title: str) -> str:
    """标题规范化：去标签、去空白、转小写"""
    if not title:
        return ""
    # 去掉常见的后缀来源标记（如 "_央广网"、"_新浪科技"）
    for suffix in ("_央广网", "_新浪科技", "_澎湃新闻", "_财联社"):
        if title.endswith(suffix):
            title = title[: -len(suffix)]
    return " ".join(title.strip().lower().split())


def deduplicate_rss(db_path: Path, threshold: float = SIMILARITY_THRESHOLD) -> int:
    conn = sqlite3.connect(db_path)
    conn.row_factory = sqlite3.Row
    cur = conn.cursor()

    # 读取当日 RSS 条目（如果表不存在则直接退出）
    try:
        cur.execute(
            """
            SELECT rowid, title, url, feed_id, rank, created_at
            FROM rss_items
            ORDER BY COALESCE(rank, 999999) ASC, created_at ASC
            """
        )
    except sqlite3.OperationalError as e:
        print(f"跳过：{db_path} 中没有 rss_items 表或字段不匹配 ({e})")
        conn.close()
        return 0

    rows = cur.fetchall()
    kept: list[sqlite3.Row] = []
    deleted_ids: list[int] = []

    for row in rows:
        norm = normalize_title(row["title"])
        if not norm:
            kept.append(row)  # 空标题保留，交给 TrendRadar 的空标题防护处理
            continue

        is_duplicate = False
        for k in kept:
            k_norm = normalize_title(k["title"])
            if not k_norm:
                continue
            ratio = SequenceMatcher(None, norm, k_norm).ratio()
            if ratio >= threshold:
                is_duplicate = True
                break

        if is_duplicate:
            deleted_ids.append(row["rowid"])
        else:
            kept.append(row)

    if deleted_ids:
        placeholders = ",".join("?" for _ in deleted_ids)
        cur.execute(f"DELETE FROM rss_items WHERE rowid IN ({placeholders})", deleted_ids)
        conn.commit()
        print(f"✅ 已从 {db_path.name} 删除 {len(deleted_ids)} 条重复 RSS 条目（阈值 {threshold}）")
    else:
        print(f"✅ {db_path.name} 未发现需要去重的 RSS 条目")

    conn.close()
    return len(deleted_ids)


def main():
    parser = argparse.ArgumentParser(description="TrendRadar RSS 标题去重补丁")
    parser.add_argument("--db", type=Path, help="指定 SQLite 数据库路径")
    parser.add_argument(
        "--threshold",
        type=float,
        default=SIMILARITY_THRESHOLD,
        help=f"标题相似度阈值，默认 {SIMILARITY_THRESHOLD}",
    )
    args = parser.parse_args()

    project_root = Path(__file__).resolve().parent.parent

    if args.db:
        db_path = args.db
    else:
        db_path = find_latest_db(project_root)

    if not db_path or not db_path.exists():
        print("错误：找不到可用的 .db 文件，请用 --db 手动指定", file=sys.stderr)
        sys.exit(1)

    deduplicate_rss(db_path, args.threshold)


if __name__ == "__main__":
    main()
