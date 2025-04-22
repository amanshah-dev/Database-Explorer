create function "SyncChapterQuestions"(chapterid integer) returns void
    language plpgsql
as
$$
BEGIN
  insert into "ChapterQuestion" ("chapterId", "questionId") 
    select "DuplicateChapter"."dupId", "ChapterQuestion"."questionId" 
      from "ChapterQuestion", "DuplicateChapter", "SubjectChapter", "Subject"
      where "ChapterQuestion"."chapterId" = "DuplicateChapter"."origId"
        and "SubjectChapter"."chapterId" = "ChapterQuestion"."chapterId" 
        and "SubjectChapter"."subjectId" = "Subject"."id"
        and "Subject"."courseId" = 8
        and "DuplicateChapter"."dupId" = chapterId
    EXCEPT 
    select "ChapterQuestion"."chapterId", "ChapterQuestion"."questionId"
      from "ChapterQuestion" 
      where "ChapterQuestion"."chapterId" = chapterId;
  
  Delete from "ChapterQuestion" where "chapterId" = chapterId and "questionId" in (select "questionId" from 
  (select "ChapterQuestion"."chapterId", "ChapterQuestion"."questionId"
    from "ChapterQuestion" 
    where "ChapterQuestion"."chapterId" = chapterId
      and EXISTS (select 1 from "DuplicateChapter" where "dupId" = chapterid)
  EXCEPT 
  select "DuplicateChapter"."dupId", "ChapterQuestion"."questionId" 
    from "ChapterQuestion", "DuplicateChapter", "SubjectChapter", "Subject"
    where "ChapterQuestion"."chapterId" = "DuplicateChapter"."origId"
    and "SubjectChapter"."chapterId" = "ChapterQuestion"."chapterId" 
    and "SubjectChapter"."subjectId" = "Subject"."id"
    and "Subject"."courseId" = 8
    and "DuplicateChapter"."dupId" = chapterid) deleteQuestions);
END
$$;

alter function "SyncChapterQuestions"(integer) owner to learner;
