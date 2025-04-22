create function compute_similarity() 
returns trigger
    security definer
    language plpgsql
as
$$
BEGIN
    -- Compute the similarity between the questions referenced by NEW."questionId1" and NEW."questionId2"
    select similarity(
        (select "question" from "Question" where "id" = NEW."questionId1"), 
        (select "question" from "Question" where "id" = NEW."questionId2")
    ) into NEW."similarity";
    
    -- Return the modified NEW record
    RETURN NEW;
END
$$;

alter function compute_similarity() owner to learner;
