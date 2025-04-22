create function "SyncChapterQuestionsCourse"(chapterid integer, courseid integer) returns void
    language plpgsql
as
$$
BEGIN
  insert into "ChapterQuestion" ("chapterId", "questionId") 
    select "DuplicateChapter"."dupId", "ChapterQuestion"."questionId" 
      from "ChapterQuestion", "DuplicateChapter", "DuplicateChapter" "DuplicateChapter1", "SubjectChapter", "Subject"
      where "ChapterQuestion"."chapterId" = "DuplicateChapter1"."dupId"
        and "DuplicateChapter1"."origId" = "DuplicateChapter"."origId"
        and "SubjectChapter"."chapterId" = "ChapterQuestion"."chapterId" 
        and "SubjectChapter"."subjectId" = "Subject"."id"
        and "Subject"."courseId" = courseId
        and "DuplicateChapter"."dupId" = chapterId
    EXCEPT 
    select "ChapterQuestion"."chapterId", "ChapterQuestion"."questionId"
      from "ChapterQuestion" 
      where "ChapterQuestion"."chapterId" = chapterId;
  
  Delete from "ChapterQuestion" where "chapterId" = chapterId and "questionId" in (select "questionId" from 
  (select "ChapterQuestion"."chapterId", "ChapterQuestion"."questionId"
    from "ChapterQuestion" 
    where "ChapterQuestion"."chapterId" = chapterId
      and EXISTS (select 1 from "DuplicateChapter" where "dupId" = chapterId)
  EXCEPT 
  select "DuplicateChapter"."dupId", "ChapterQuestion"."questionId" 
    from "ChapterQuestion", "DuplicateChapter", "DuplicateChapter" "DuplicateChapter1", "SubjectChapter", "Subject"
    where "ChapterQuestion"."chapterId" = "DuplicateChapter1"."dupId"
    and "DuplicateChapter1"."origId" = "DuplicateChapter"."origId"
    and "SubjectChapter"."chapterId" = "ChapterQuestion"."chapterId" 
    and "SubjectChapter"."subjectId" = "Subject"."id"
    and "Subject"."courseId" = courseId
    and "DuplicateChapter"."dupId" = chapterId) deleteQuestions);
END
$$;

alter function "SyncChapterQuestionsCourse"(integer, integer) owner to learner;
