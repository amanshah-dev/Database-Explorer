create function "SetChapterFlashCardSeq"(forcerefresh boolean DEFAULT false) 
    returns void 
    language plpgsql 
as 
$$
BEGIN
  IF forceRefresh THEN 
    UPDATE "ChapterFlashCard" c
      SET "seqId" = c2.seqnum
      FROM (
        SELECT c2."id", row_number() over (PARTITION BY "chapterId" ORDER BY "flashCardId") as seqnum
        FROM "ChapterFlashCard" c2
      ) c2
      WHERE c2."id" = c."id";
  ELSE 
    UPDATE "ChapterFlashCard" c
      SET "seqId" = c2.seqnum
      FROM (
        SELECT c2."id", row_number() over (PARTITION BY "chapterId" ORDER BY "flashCardId") as seqnum
        FROM "ChapterFlashCard" c2
      ) c2
      WHERE c2."id" = c."id" 
        AND c."seqId" IS NULL;
  END IF;
END;
$$;

alter function "SetChapterFlashCardSeq"(boolean) owner to learner;
