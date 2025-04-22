create function "SetChapterFlashCardSeq"(chapterid integer) 
    returns void 
    language plpgsql 
as 
$$
BEGIN
  UPDATE "ChapterFlashCard" c
    SET "seqId" = c2.seqnum
    FROM (
        SELECT c2."id", row_number() over (order by "flashCardId") as seqnum
        FROM "ChapterFlashCard" c2 
        WHERE "chapterId" = chapterid
    ) c2
    WHERE c2."id" = c."id";
END;
$$;

alter function "SetChapterFlashCardSeq"(integer) owner to learner;
