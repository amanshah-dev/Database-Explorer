
create procedure update_synapse1expert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Synapse I', achievementCategoryId, userIds);
END;
$$;

alter procedure update_synapse1expert(bigint, bigint[]) owner to learner;
